# Browser Shutdown, how does it work? #

## General Overview
- The parent process goes through a number of observer notifications that
  trigger different phases of shutdown.  Traditionally, each observer would spin
  its own nested event loop until it was satisfied shutdown had been completed,
  resulting in serialized shutdown with a lot of potential inefficiency.  We're
  now moving to using the AsyncShutdown mechanism which tells all its consumers
  about a phase in parallel and then spins a single event loop until they all
  report shutdown completion.  If they don't do so in a timely fashion, a crash
  is induced and the crash reporter indicates what shutdown blockers were still
  outstanding, making it easier to figure out who to yell at.
  - JS consumers should consult https://searchfox.org/mozilla-central/source/toolkit/components/asyncshutdown/AsyncShutdown.jsm
  - C++ consumers should consult https://searchfox.org/mozilla-central/source/toolkit/components/asyncshutdown/nsIAsyncShutdown.idl
- For Content Processes, most observed shutdown behavior will happen because
  when PContent is torn down this will destroy all the PBrowser instances, which
  will in turn destroy the windows/documents.  Prior to this, some very minimal
  observer notifications will run that are designed for when cleanup is
  essential and can't be handled like the process crashed.  After  this, things
  diverge based on build type:  
  - Under release builds, both the parent and the child start timers to
    terminate the child, currently set to 5 seconds.  But unless the process is
    badly unresponsive, this is unlikely to matter because
    ContentChild::ActorDestroy will immediately terminate the process via
    exit(0).  This will happen after all of its child actors are destroyed.
  - Under debug builds, there are no termination timers and there is no exit(0),
    and since PBackground is still alive, most PBackground services will see
    an orderly shutdown of their clients.  The full set of observer
    notifications will also

## IPC Details
This just deals with parent/child stuff.  There's a gazillion shutdown handlers
in the parent.

### Content Process Shutdown

#### ContentParent
- each ContentParent::Observe receives "profile-per-change" and:
  - Sends a "Shutdown" message to its ContentChild via
    `ShutDownProcess(SEND_SHUTDOWN_MESSAGE);` then spins a nested event loop
    until IPC has closed or our timeout mechanism caused us to forcibly kill the
    child.  The loop looks like:
    `SpinEventLoopUntil([&]() { return !mIPCOpen || mCalledKillHard; });`
    - ShutDownProcess will also invoke StartForceKillTimer() which creates a
      timer that waits "dom.ipc.tabs.shutdownTimeoutSecs" seconds before
      invoking ContentParent::KillHard().  Complications:
      - In the presence of an XPCSHELL_TEST_PROFILE_DIR env var that suggests
        xpcshell tests, KillHard never gets invoked no matter what.
      - Under debug builds, the pref is set to 0 which causes the timer to not
        be initialized.  This helps verify clean shutdown because there are
        still other shutdown timers in play that will trigger an abort.
      - Under non-(debug/ASAN/TSAN/Valgrind) builds, the value is set to 5
        seconds, which is also the in-code default.
  - While in that loop:
    - A "FinishShutdown" message will be received.  This will invoke
      `ShutDownProcess(CLOSE_CHANNEL)` which will:
      - QuotaManagerService::AbortOperationsForProcess(mChildID) will be
        invoked.


#### ContentChild
The below happens for each child.
- ContentChild receives a "Shutdown" message.
  - A "content-child-will-shutdown" observer notification is generated.  This
    was added in https://bugzilla.mozilla.org/show_bug.cgi?id=1425559 to surface
    shutdown to the new nsThreadManager::SpinEventLoopUntilOrShutdown(condition)
    method so that modal dialogs and other event-loop spinners can know to
    terminate at shutdown.
    - Note that ContentChild::IsShuttingDown() also exists if you're thinking
      about observing for this notification, but that it won't return true until
      all nested event loops have been popped.
  - CrashReporter::AnnotateCrashReport logs the "IPCShutdownState" as
    RecvShutdown.
  - ContentChild::ShutdownInternal() is invoked which covers the rest of the
    logic.
  - If a nested event loop is detected (they won't have unwound yet), a 100ms
    backoff will repeatedly be tried before advancing.  It's ShutdownInternal
    that gets rescheduled, so the previous steps won't get repeated.
  - A "content-child-shutdown" observer notification is generated.  (Contrast
    with the earlier WILL shutdown message.)
    - TelemtrySession.jsm does Telemetry.flushBatchedChildTelemetry().
    - ContentObservers.js removes its WebRTC observers that would otherwise be
      relayed to ContentWebRTC.jsm, presumably with some kind of overhead.
  - On XP mozilla::widget::StopAudioSession() is invoked for some reason.
  - The PContent IPC channel is set to not abort on error.  (Where abort means
    crash the process.)
  - The profiler is told to send any profiling results and shutdown if it was
    compiled in and active.
  - The child force kill timer is activated.  This uses the same
    "dom.ipc.tabs.shutdownTimeoutSecs" pref as the parent, but without the
    xpcshell carve-out.  The pref is 0, disabling the kill timer, on non-debug
    builds.
  - A "FinishShutdown" message is sent to the parent.
- The PContent channel will be closed by the parent, resulting in PContent being
  torn down and ContentChild::ActorDestroy being invoked which will:
  - If in a release build, ProcessChild::QuickExit() which is equivalent to
    exit(0) will be invoked, self-terminating the process with a success error
    code.  Stop reading now, nothing more will ever happen in this case.
  - Various small cleanup stuff happens:
    - The ConsoleListener that propagates console message to the parent via
      PContent "ScriptErrorWithStack" or "ScriptError" messages is removed.
  - XRE_ShutdownChildProcess is invoked which:
    - The Scheduler is shutdown.  This is the thing that maintains a bunch of
      main-thread priority queues and existed for the now doomed cooperative
      scheduling effort.
    - MessageLoop::Quit() is invoked which will set the quite_received flag so
      that the next time MessageLoop::DoIdleWork is invoked and does not have
      anything to do, it will invoke MessagePump::Quit.  For the main thread,
      DoIdleWork will only be invoked by MessagePump::Run once
      NS_ProcessNextEvent(, aMayWait=false) returns false indicating that there
      were no events processed.  This means that we wait for the normal XPCOM
      main-thread event target to become empty, plus the IPC and IPC idle queues
      to drain as well.  Once that happens, keep_running_ gets set and
      MessagePump::Run will be allowed to terminate.  (And this must happen
      after the ActorDestroy call returns because it's mechanisms above this on
      the stack.)
- Once MessagePump::Run terminates, we return this this callstack:
  - MessagePumpForChildProcess::Run
  - XRE_InitChildProcess which will invoke ContentProcess::Cleanup which invokes
    XRE_TermEmbedding which does:
    - nsXREDirProvider::DoShutdown which does all the observer notifications,
      see https://searchfox.org/mozilla-central/rev/c0d81882c7941c4ff13a50603e37095cdab0d1ea/toolkit/xre/nsXREDirProvider.cpp#1059
      noting that this is guarded by mProfileNotified which only triggers and
      induces startup notifications when preferences are received from the
      parent process.
      - "profile-change-net-teardown"
      - "profile-change-teardown"
      - "profile-before-change"
      - "profile-before-change-qm"
      - "profile-before-change-telemetry"
    - NS_ShutdownXPCOM is invoked which invokes mozilla::ShutdownXPCOM which is
      a general shutdown that's identical to what happens in the parent, but I
      went and started transcribing stuff below because I love observer
      notifications:
      https://searchfox.org/mozilla-central/source/xpcom/build/XPCOMInit.cpp#801
      - sends observer notifications:
        - "xpcom-will-shutdown"
        - "xpcom-shutdown"
        - "xpcom-shutdown-threads"
      - marks threads as shutting down
      - nsTimerImpl::Shutdown()
      - actual thread shutdown via nsThreadManager::Shutdown()
      - "xpcom-shutdown-loaders" observers are saved off
      - the observer service is shutdown
      - mozilla::services::Shutdown() is invoked
      - nsComponentManagerImpl::FreeServices()
      - "xpcom-shutdown-loaders" is manually dispatched from the saved-off
        enumerator.
      - nsCycleCollector_shutdown(shutdownCollect); where shutdownCollect is
        true if DEBUG/TSAN/ASAN/etc. or run if the "MOZ_CC_RUN_DURING_SHUTDOWN"
        env var is present under release/non-debug.
      - nsComponentManagerImpl::Shutdown() is invoked
      - JS_ShutDown()
      - NSS_Shutdown()
      - TODO: finish analysis/put this generic bit someplace more sensible.
  - content_process_main
