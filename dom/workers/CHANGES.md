# Detailed Change Dumps #

Where review notes go, or fresh patch investigations that impact something.

### Bug 1263304 : SW async waitUntil ###
https://bugzilla.mozilla.org/show_bug.cgi?id=1263304

#### Move Service Worker MessageEvent dispatching code to ServiceWorkerPrivate. ####
https://hg.mozilla.org/mozilla-central/rev/85a6e5e89b42

ServiceWorkerPrivate.cpp:
* MessageWaitUntilHandler removed
* SendMessageEventRunnable (isa ExtendedEventWorkerRunnable) introduced
* SWP::SendMessageEvent shim that called out to
  WorkerPrivate::PostMessageToServiceWorker refactored to use the new
  SendMessageEventRunnable.

WorkerPrivate.cpp:
* ServiceWorker-specific bits removed from MessageEventRunnable.
* WorkerPrivate::PostMessageToServiceWorker and call plumbing removed.

#### Allow waitUntil() to be called asynchronously. ####
https://hg.mozilla.org/mozilla-central/rev/509e98eec0ce

ServiceWorkerEvents.cpp:
* ExtendableEvent gained WaitOnPromise(Promise&)
* ExtensionsHandler nested subclass/interface introduced.
  * ExtendableEvent has mExtensionsHandler, set by SetKeepAliveHandler.
(in anon namespace:)
* ExtendableEventResult (non-class) enum: Rejected=0, Resolved
* ExtendableEventCallback refcounted single-method class with
  typed FinishedWithResult(ExtendableEventResult) callback.
* KeepAliveHandler (isa WorkerHolder, ExtensionsHandler, PromiseNativeHandler)
  * holds mKeepAliveToken like predecessors
  * edge-case support: holds self-reference for unreachable promise we're
    waiting on indefinitely (with comment! :)
  * tracks mPendingPromisesCount, not the specific promises.
  * tracks mRejected
  * ResolvedCallback/RejectedCallback call RemovePromise(ExtendableEventResult)
    * Asserts on worker thread (these were promises in the SW global)
    * If mPendingPromisesCount hits zero, rather than invoking MaybeDone
      immediately, NewRunnableMethod(MaybeDone) is put in the microtask queue
      so that we wait a turn of the microtask queue (as spec'ed).
  * MaybeDone:
    * bails if mPendingPromisesCount went >0 again.
    * invokes mCallback->FinishedWithResult(mRejected ? Rejected : Resolved)
  * MaybeCleanup: worker-thread-asserted, mKeepAliveToken as reentrancy guard,
    releases mKeepAliveToken, mSelfRef.
(public again:)
* DispatchExtendableEventOnWorkerScope directly invokes MaybeDone after dispatch
  so that, per spec, extensions are disallowed without microtask queue turns if
  there was no initial waitUntil.
* Converted to ExtendableEventCallback from PromiseNativeHandler:
  * LifeCycleEventWatcher
  * PushErrorReporter
  * AllowWindowInteractionHandler
