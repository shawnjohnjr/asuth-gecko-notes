## PENDING SUPER IMPORTANT TO-RESOLVE BITS

- pre-allocated process.  We talked about just not having the service register
  itself until the process explicitly gets consumed / has its type set.
- the freeze/thaw logic is still broken.

## Overall Restating:

### In-tree documentation

- dom/workers/remoteworkers/RemoteWorkerController.h
  - Awesome ASCII art communication diagram with labeled/explained steps.
    Captures all the moving parts concisely.

### Class overviews and relations:
The order here is roughly the order in which classes will be involved as a
SharedWorker is created.

- SharedWorker: DOM binding.  Holds a SharedWorkerChild.  Must exist on the main
  thread because we only allow top-level windows to create SharedWorkers.
- SharedWorkerChild: Held by SharedWorker bindings to remotely control
  sharedworker lifecycle and receive error and termination reports.
- PSharedWorker: Protocol for SharedWorker bindings to communicate with
  per-worker SharedWorkerManager instances in the parent via SharedWorkerChild/
  SharedWorkerParent and SharedWorkerService getting/creating the
  SharedWorkerManager if it doesn't already exist.  Main-thread to PBackground.
- SharedWorkerParent: PBackground actor that relays life-cycle events
  (freeze/thaw, suspend/resume, close) to the PBackground SharedWorkerManager
  and relays error/termination back to the child.
- SharedWorkerService: PBackground service that creates and tracks the
  per-worker SharedWorkerManager instances, allowing rendezvous between
  SharedWorkerParent instances and the SharedWorkerManagers they want to talk to
  (1:1).
- SharedWorkerManager: PBackground instance that corresponds to a single logical
  Shared Worker that exists somewhere in the process tree.  Referenced/owned by
  multiple SharedWorkerParent instances on the PBackground thread.  Holds/owns
  a single RemoteWorkerController to interact with the actual shared worker
  thread, wherever it is located.  Creates the RemoteWorkerController via
  RemoteWorkerController::Create which uses RemoteWorkerManager::Launch under
  the hood.

- RemoteWorkerController: PBackground instance created by static
  RemoteWorkerController::Create that builds on RemoteWorkerManager.  Interface
  to control the remote worker as well as receive events via the
  RemoteWorkerObserver interface that the owner (SharedWorkerManager in this
  case) must implement to hear about errors, termination, and whether the
  initial spawning succeeded/failed.  Its methods may be called immediately
  after creation even though the worker is created asynchronously; an internal
  operation queue makes this work.  Communicates with the remote worker via
  owned RemoteWorkerParent over PRemoteWorker protocol.
- RemoteWorkerManager: PBackground instance that keeps tracks of
  RemoteWorkerServiceParent actors (1 per process, including the main process)
  and pending RemoteWorkerController requests to spawn remote workers if the
  spawn request can't be immediately fulfilled.  Decides which
  RemoteWorkerServerParent to use internally via SelectTargetActor in order to
  select a BackgroundParent manager on which to create a RemoteWorkerParent.
- RemoteWorkerService: Every process has a RemoteWorkerService which does the
  actual spawning of RemoteWorkerChild instances.  The RemoteWorkerService
  creates a "Worker Launcher" thread at initialization on which it creates a
  RemoteWorkerServiceChild to service spawn requests.  The thread is exposed as
  RemoteWorkerService::Thread().  A new/distinct thread is used because we
  (eventually) don't want to deal with main-thread contention, content processes
  have no equivalent of a PBackground thread, and actors are bound to specific
  threads.  (Disclaimer: currently most RemoteWorkerOps need to happen on the
  main thread because the main-thread ends up as the owner of the worker and
  all manipulation of the worker must happen from the owning thread.)
- RemoteWorkerServiceChild: "Worker Launcher"-thread child actor created by the
  RemoteWorkerService to register itself with the PBackground
  RemoteWorkerManager in the parent.
- RemoteWorkerServiceParent: PBackground parent actor that registers with the
  PBackground RemoteWorkerManager and used to relay spawn requests.
- RemoteWorkerParent: PBackground-managed parent actor that is mutually
  associated with a single RemoteWorkerController.  Relays error/close events to
  the controller and in turns is told life-cycle events.
- RemoteWorkerChild: PBackground-managed "Worker Launcher"-thread-resident
  created via the RemoteWorkerManager to actually spawn the worker.  Currently,
  the worker will be spawned from the main thread due to nsIPrincipal not being
  able to be created on background threads and other ownership invariants, most
  of which can be relaxed in the future.

### RemoteWorker API surface exposed to SharedWorkers / ServiceWorkers

Just RemoteWorkerController::Create.  Very straightforward!

### General Mechanics
- IPC ownership idiom is sometimes refcounted with manual forget().take() on
  alloc and transfered into a RefPtr with dont_AddRef (that will release the
  refcount on destruction at scope exit) on dealloc.
  - Actor life tracking:
    - mActive: SharedWorkerChild
    - mIPCActive: RemoteWorkerChild
  - mActive tracks when the actor has been destroyed.
    So far: **SharedWorkerChild**
- All communication always goes through PBackground, even if co-located in
  the same process.  This helps keep ordering guarantees sane.

- RemoteWorkerData (initially SharedWorkerLoadInfo) in
  dom/workers/remoteworkers/RemoteWorkerTypes.ipdlh characterizes the necessary
  spawning LoadInfo.
  - Contains:
    - The SharedWorker's name as given to `new SharedWorker(name)`.
    - Relevant URL's (original script, base script, resolved script).
    - Loading principal and principal, encoded as PrincipalInfos because it's a
      PBackground protocol.
    - Custom serialized representations of the CSP and preload CSP for each
      principal to be re-attached to the Principal.
    - The domain.
    - isSecureContext: The result of running the algorithm specificed at
      https://w3c.github.io/webappsec-secure-contexts/#is-settings-object-contextually-secure.
      This is different than the more straightforward "is origin potentially
      trustworthy?" check from the same spec that contextually secure also
      depends on.  In particular, an https page in an iframe controlled by a
      non-https page will not be a secure context.
      - There is a single namespace for SharedWorkers shared between pages for
        which this is true and pages where it is false.  The worker does not get
        keyed by the secure context, so in a case where there's an unframed
        https page and an insecurely framed https page, there's a race for who
        gets to create the SharedWorker that starts and lives.  Compare with the
        persistent storage in IndexedDB/etc. which will be the same either way.
        Compare again with ServiceWorkers which only deal with secure contexts
        (although the spec does use 'potentially trustworthy URL' as a fallback
        when the client is not an environment settings object, which I think is
        maybe just for notification image loading?).
    - **clientInfo**
    - isSharedWorker **forward looking**
  - The information is populated by SharedWorker in its constructor and sent
    with the creation request.
  - SharedWorkerManager::MatchOnMainThread is what checks the re-hydrated info
    to determine whether an existing shared worker corresponds.  This is only
    a check of domain, resolved script URL, name, and mutual loading principal
    subsumption.  `isSecureContext` is checked subsequently to generate an error
    if there's a mismatch per spec.  (Compare with also keying on the secure
    context.)
    - The loading principal is used so that two data URI's from the same origin
      can still use SharedWorkers.  This is covered by the language "constructor
       origin" in step 11.2 of SharedWorker construction in the HTML spec.
    - Mutual subsumption is somewhat of a paranoia thing given the domain
      check.  The general concern is that the system principal subsumes
      everything and an expanded principal (as used by web extension frame
      scripts) subsumes less-privileged content principals.  It would be very
      bad if SharedWorkers allowed a content privileged principal to connect to
      a more powerful principal's SharedWorker.

- SharedWorker remains the binding class.  But now instead of dealing with
  WorkerPrivate directly, it deals with SharedWorkerChild.
- SharedWorkerChild is the handle class to reference the actual RemoteWorker
  that may live in a different process.


- SharedWorkerService: SharedWorkerManager rendezvous mechanism.  A PBackground
  service that keeps track of SharedWorkerManagers on the main-thread because
  nsIPrincipal is main-thread-only.
  - General Usage:
    - SharedWorkerParent instances invoke GetOrCreateWorkerManager on
      PBackground thread to asynchronously trigger SharedWorkerManager creation.
      - Completion notification via SharedWorkerParent::ManagerCreated.  The
        actor is automatically registered with the manager, but responsible for
        removing itself in ActorDestroy/RecvClose/ManagerCreated.
      - Async because of the need to bounce to the main thread for nsIPrincipal.
      - A single list of mWorkerManagers is checked using
        SharedWorkerManager::MatchOnMainThread, see RemoteWorkerData notes
        above.
      - If there was no existing manager, a new SharedWorkerManager is created.

  - Lifecycle: Controlled via refcount, not a singleton with shutdown logic.
    - Synchronously created on background thread via GetOrCreate.
    - At-most-once instance is handled by sSharedWorkerService weak pointer
      with sSharedWorkerMutex guarding access to sSharedWorkerService from
      GetOrCreate(), Get(), and the destructor (which clears).
    - Goes away when the last strong refcount goes away.  Conceptually, this
      should be due to the last SharedWorkerParent instance going away and
      dropping its refcount, but



- SharedWorkerManager: 1 per worker thread.

- SharedWorkerParent states:
  - eInit: Initial state, extremely short-lived, should progress to ePending
    during same turn of event loop as RecvPSharedWorkerConstructor is invoked.
  - ePending: State until ManagerCreated callback is invoked.  As far as IPC
    is concerned, same as eActive, but calls to mWorkerManager need to be
    deferred until ManagerCreated.  (Just mFrozen/mSuspended, simple booleans.)
  - eActive: State once ManagerCreated is invoked until RecvClose is received.
    SharedWorker can trigger its Close() method because of:
    - Destruction
  - eClosed:

- RemoteWorkerService:
  - Startup - All Processes:
    - RemoteWorkerService::Initialize() invoked depending on process:
      - Parent: By nsLayoutStatics::Initialize(), the same place we initialize
        DOMPrefs. This happens from NS_InitXPCOM2() which is before
        nsXREDirProvider::DoStartup() when profile-do-change/profile-after
        change will be invoked.  What happens in here varies by parent/child.
      - Child: By ContentChild::InitXPCOM() after ClientManager::Startup().
      - StaticRefPtr sRemoteWorkerService strong reference initialized here from
        RemoteWorkerService::Initialize() through explicit Shutdown process.
        (Destruction is NOT shutdown.)
    - RemoteWorkerService::InitializeOnMainThread() happens next.  For the
      parent, this only happens after "profile-after-change" fires.  For child
      processes, it happens during Initialize().
      - A new thread, "Worker Launcher" is created.  It creates a
        RemoteWorkerServiceChild (PBackground-managed) actor.
      - InitializeOnTargetThread is scheduled to run on the thread,
        (synchronously) creating the PBackground actor, then creating the
        PRemoteWorkerService actor. which is held in mActor.
  - Shutdown - All Processes
    - Happens at "xpcom-shutdown".  For content processes, this is something
      that only happens in DEBUG/similar builds because the content processes
      will exit() prior to that in normal builds.  However, it will definitely
      happen in the parent process.
      - A ShutdownOnTargetThread() function runnable gets dispatched to the
        "Worker Launcher" thread and sRemoteWorkerService gets cleared (with the
        mutex held).
      - ShutdownOnTargetThread() issues the actor deletion (from the correct
        thread, as is necessary) which is proper because in this DEBUG case,
        PBackground will still be around.  A lambda is then bounced back to the
        main thread to trigger thread shutdown with the label
        "ShutdownOnMainThread".

- RemoteWorkerController:
  - State transitions:
    - ePending: Initial
    - ePending -> eReady: CreationSucceeded
    - ePending -> eTerminated: Shutdown
    - eReady -> eTerminated: Shutdown
  - Lifecycle notables:
    - Creation:
      - Drain mPendingOps via manual invocation of the op API methods again
        based on big switch.
        - Ops are marked completed so that the op constructor doesn't have to
          do failsafe stuff such as nuke the port.
    - Shutdown:
      - Controller mutual pointers nulled out.
      - mPendingOps cleared: **x**
      - RemoteWorkerTerminateOp sent.
  - General op exposed API structure:
    - If ePending, queue in mPendingOps
    - If eTerminated, fast return.
    - Otherwise issue actual send.


- RemoteWorkerParent:

- RemoteWorkerChild
  - WorkerState states:
    - ePending: "CreationSucceeded/CreationFailed not called yet."
    - ePendingTerminated: "The worker is not created yet, but we want to terminate
      as soon as possible.
    - eRunning: "Worker up and running."
    - eTerminated: "Worker terminated."

- Error propagation:
  - PSharedWorker.Error


## Devtools console issues


## Specific Patch Notes

### Part 2: PSharedWorker protocol
Already reviewed, see previous review tracking below.

- SharedWorkerParent/Child actors introduced.
- Protocol introduced.
- SharedWorkerChild replaces WorkerPrivate in SharedWorker.cpp.

### Part 3: SharedWorkerService and SharedWorkerManager
Already reviewed, see previous review tracking below.

- WorkerLoadInfo gains mSecureContext an {eNotSet,eInsecureContext,
  eSecureContext} enum that needs to exist because WorkerLoadInfo's are
  constructed uninitialized and only have StealFrom for bulk initialization.
- BackgroudParentImpl implements PSharedWorkerConstructor to invoke Initialize.
-
- Makes nsGlobalWindowInner track mSharedWorkers explicitly.
  - Adds StoreSharedWorker/ForgetSharedWorker.  **could these be DETH-style or
    bkelly's lifecycle analog?**
  - Tracked using nsTObserverArray, which is good hygiene, but none of the calls
    allow for JS to be invoked on the stack and so should not cause coordination
    issues.
- RuntimeService has prev-patch commented-out blocks related to SharedWorker
  management (UnregisterWorker,CancelWorkers*,etc.) removed, and new assertions
  added.
- **Q: where'd the "error" event for secureContext mismatch go?**  Is it even
  needed now that things are partitioned via OriginAttributes anyways?  **so
  check on storage map/list changes**.  Previously, mDomainMap was keyed by
  WorkerLoadInfo->mDomain.  The old secureContext check at least was based on
  WorkerPrivate and the calling context's compartment.  Not sure how someone
  would escape origin attributes there.
- SharedWorker::ErrorPropagation(nsresult) helper added which creates an
  AsyncEventDispatcher that posts, and then invokes Close().

### Part 4:
Already reviewed, see previous review tracking below.

### Part 5:
Already reviewed, see previous review tracking below.

### Part 6: CSP via IPCz
Already reviewed by ckershb.

### Part 7: SharedWorker can be intercepted by a ServiceWorker

Adds a mechanism to WorkerLoadInfo to set an mOuterRequestor which is set during
CreateWorkerOnMainThread and is an nsIInterfaceRequestor implementing
GetInterface for nsINetworkInterceptController.

Moves to RemoteWorkerChild in Part 9.

### Part 8: RemoteWorker IPC
Status: marked all but the following as reviewed:
- RemoteWorkerController.{h,cpp}
- RemoteWorkerManager.{h,cpp}
- RemoteWorkerParent.{h,cpp} although I think this has been largely audited in
  general mechanics above, so it's just a post-part-8 control flow sanity
  check that's necessary.

Questions mooted by proposal on bug:
- How do we avoid weirdness with the preallocated process?
  - So, the launch command does use the getorcreate call that will steal the
    preallocated process.  But it won't be used if it's already been registered?
  -


NB: See "General Mechanics" above for high-level.

Startup:
  - RemoteWorkerService::Initialize call added to nsLayoutStatics in parent
    process to be immediately initialized after DOMPrefs::Initialize.

IPC:
- PRemoteWorkerService: PBackground-managed, link established from child to
  parent (via constructor over PBackground).
  - Parent: not refcounted.
  - Child: refcounted.

- PRemoteWorker: PBackground-managed, sent from parent-to-child (via constructor
  over PBackground link)
  - Parent: uses refcounted idiom.
  - Child: not refcounted
  - Constructor payload is RemoteWorkerData, initially empty.

RemoteWorkerService:


### Part 9: RemoteWorker in SharedWorkerManager

meta: bunch of consts added.

PRemoteWorker.ipdl:
- Bunch of per-op structs added under RemoteWorkerOp union.
- to-parent Created() gained a bool aStatus arg.
- to-parent Error(ErrorValue), Close() added.
- to-child: ExecOp(RemoteWorkerOp) added.

RemoteWorkerTypes.ipdlh refactored to include much of the deleted
SharedWorkerTypes.ipdlh:
- ContentSecurityPolicy moved over
- SharedWorkerLoadInfo's contents largely moved into RemoteWorkerData save for:
  - windowID and MessagePortIdentifier are now separate arguments in the
    PBackground constructor for PSharedWorker instances.
  - RemoteWorkerData gains `isSharedWorker` to distinguish from ServiceWorker.
- ErrorDataNote/ErrorData/ErrorValue all migrated over.

RemoteWorkerChild.h:
- Gains RefPtr<WeakWorkerRef> mWorkerRef (which just notices if the worker dies)
- Also holds RefPtr<WorkerPrivate> mWorkerPrivate
- Gains WorkerState ePending/ePendingTerminated/eRunning/eTerminated enum that
  is tracked in mWorkerState that's touched on the owning (Worker Launcher)
  thread only.
- Gains nsTArray<RemoteWorkerOp> mPendingOps.
- Gains nsTArray<uint64_t> mWindowIDs that's allegedly main-thread only.
RemoteWorkerChild.cpp:
- CSP application logic that spreads nsTArrays of ContentSecurityPolicy
  aPolicies and aPreloadPolicies onto the nsIContentSecurityPolicy pulled off
  the principal via EnsurePreloadCSP.
- SharedWorkerInterfaceRequestor moved here from SharedWorkerManager.
- MessagePortIdentifierRunnable WorkerRunnable added to service
  RemoteWorkerPortIdentifierOp to be sent from worker launcher thread to the
  WorkerPrivate thread where it invokes RWC::AddPortIdentifier which invokes
  ConnectMessagePort.
  - ConnectMessagePort creates the MessagePort binding for the global, entangles
    it, and dispatches the event.
- ExecWorker is responsible for creating the actual worker, and is called by
  BackgroundChildImpl::RecvPRemoteWorkerConstructor.
- ExecWorkerOnMainThread is the actual worker spawning logic.  Notable things:
  - The mStorageAllowed check is `access > StorageAccess::ePrivateBrowsing`
    which is notably the same as ServiceWorkerPrivate::SpawnWorkerIfNeeded
    rather than WorkerPrivate's default which is `> StorageAccess::eDeny.`
    - ePrivateBrowsing is 1 and eDeny is 0.  So this notably forbids storage
      access to private browsing..


RemoteWorkerController.h:
- private struct Op helper:
  - enum Type has values corresponding to PRemoteWorker.ipdl op structs.
  - state is Type; plus the non-unioned/non-variant payload types:
    MessagePortIdentifier, windowid; plus the completion tracking in a bool.
    (Variant probably would be preferable...)
  - non-copyable.
- nsTArray<UniquePtr<Op>> mPendingOps holds the ops.
RemoteWorkerController.cpp:
- RWC::Create gains the RemoteWorkerData payload argument, which previously was
  just locally intialized in the method (and so now passed down using existing
  plumbing.)
- CreationFailed() fails over to Shutdown() and has a TODO to report something.
- CreationSucceeded gains an eTerminated fast-return


WorkerPrivate:
- s/SharedWorkerManager/RemoteWorkerController/.  Set/Get changed, and
  mSharedWorkerManager is now mRemoteWorkerController.

RemoteWorkerManager:
- GetOrCreate: Asserts background thread, parent process.  Constructor helps
  ensure singleton.
- RegisterActor(RemoteServiceWorkerParent): This is a content process reporting
  in for the ability to spawn workers.  Requests to RWM::Launch where there were
  no existing actors will have queued their request into mPendings (with sketchy
  refcount boosting) and triggered a content process launch via
  RWM::LaunchNewContentProcess.
- Launch: Goes direct to LaunchInternal if there's a RemoteWorkerService actor
  available, otherwise mPendings gets involved with addref/release shenanigans
  and invokes LaunchNewContentProcess with expected async resolution via
  RegisterActor.
-

SharedWorker creation plumbing changes:
- PBackground.ipdl: SharedWorkerLoadInfo changed to RemoteWorkerData, and the
  windowID and MessagePortIdentifier added.
- Background{Child,Parent}Impl.{h,cpp}: updated consistent with the above
- SharedWorkerService.{h,cpp}: ditto, with GetOrCreateWorkerManagerRunnable and
  WorkerManagerCreatedRunnable gaining the args.

SharedWorkerService.cpp:
- SharedWorkerManager::MaybeCreateRemoteWorker added, actually triggering the
  creation of the remove worker.

SharedWorkerManager.{h,cpp}:
- Gained the new MaybeCreateRemoteWorker method used by SharedWorkerService.
- Accordingly, the same-process spawning logic is removed that was used by
  CreateWorkerOnMainThread that was invoked by
  GetOrCreateWorkerManagerOnMainThread.
- AddActor gains the "thaw" logic that previously was in
  SharedWorkerService::GetOrCreateWorkerManagerOnMainThread. **XXX** check move.
- UpdateSuspend/UpdateFrozen updated to use remoting.
- SharedWorkerManager::BroadcastErrorToActorsOnMainThread commented out in a
  TODO.  Fixed in part 10.


Fallout of SharedWorkerManager to WorkerController transition.  These all look
like going from `workerPrivate->GetSharedWorkerManager()->Foo()` to
`workerprivate->GetRemoteWorkerController()->Foo()`.
- Fetch.cpp: FlushReportsToActorsOnMainThread -> FlushReportsOnMainThread.
- RuntimeService: CloseActorsOnMainThread() -> CloseWorkerOnMainThread.
- WorkerPrivate: BroadcastErrorToActorsOnMainThread ->
  ErrorPropagationOnMainThread
- WorkerError: same

### Part 10: RemoteWorkerObserver

SharedWorkerManager.cpp:
- Dead error reporting and flushing logic code removed.
- Dead CloseActorsOnMainThread removed.  (Mooted in RuntimeService.cpp in part 9
  when the logic was changed to use the controller instead of the manager.)

RemoteWorkerController.h:
- RemoteWorkerObserver added to provide creation results and error/termination
  reporting.

### Part 11: selection of RemoteWorker actors

SharedWorkerService.{h,cpp}:
- base::ProcessId of the originating process added to MaybeCreateRemoteWorker
SharedWorkerService.cpp:
- OtherPid of

### Part 12: Spawning a new process if needed.

RemoteWorkerManager::
- Blindly invokes ContentParent::GetNewOrUsedBrowserProcess.

### Part 13: Keeping ContentProcess alive
Restating:
- The number of RemoteWorkerActors alive in each ContentParent now gets tracked
  and causes ShouldKeepProcessAlive() to indicate that the process should be
  kept alive.
- The unusual race to worry about is DEBUG build shutdown time when PContent is
  torn down but PBackground is still around and the content process goes through
  an orderly XPCOM-style shutdown for leak-tracking and assertion-y purposes.
  In release builds, content processes will exit() and there's no race.
  - In this situation, ContentParent references will still exist, including by
    the BackgroundParent which holds the PContent reference until its own actor
    is destroyed.
  - Since RegisterRemoteWorkerActor() does not convey success (it's void), and
    the worker stuff doesn't otherwise pay attention, the interesting Q is what
    happens when UnregisterRemoteWorkerActor comes around.  The good news is
    it's all fine because all the calls check mShutdownPending and idempotently
    bail.  (That said, ContentProcessManager::GetTabParentCountByProcessId will
    take an NS_WARN_IF path because ContentProcessManager::RemoveContentProcess
    is invoked by ContentParent::ActorDestroy, so the table entry will no longer
    exist.)

ContentParent:
- Gains an Atomic<uint32_t> mRemoteWorkerActors counter.
  - Causes ContentParent::ShouldKeepProcessAlive to return true as long  as the
    worker count is truthy.
- RegisterRemoteWorkerActor/UnregisterRemoteWorkerActor increment/decrement it.
  - Register is a blind increment.
  - Unregister basically does the NotifyTabDestroyed logic if
    mRemoteWorkersActors goes to zero.  Note that ShouldKeepProcessAlive is not
    enough on its own because it does not include the ContentProcessManager
    tab-count check (because NotifyTabDestroyed does that itself before calling
    ShouldKeepProcessAlive) or the (mutating by stealing) hand-off to the
    PreallocatedProcessManager.

RemoteWorkerParent:
- Gains Initialize(), invoked by RemoteWorkerManager::LaunchInternal, which
  conditionally invokes the register function if there was a ContentParent.
  (There is no ContentParent if the PBackground child actor is in the same process,
  which means for non-e10s or Chrome-privileged client code.)
  - There is also a ProxyRelease dance because ContentParent isn't thread-safe
    and BackgroundParent::GetContentParent tells a white lie by handing out an
    already_AddRefed ContentParent that uses a runnable to AddRef on the main
    thread.  The add/release aren't pedantically paired correctly, but it should
    be fine.  I've commented on the review.
- Unregister happens at ActorDestroy.  GetContentParent() is used again, but
  this time the ContentParent ref is passed-off.  This is a slightly more
  interesting situation because there's a chance of this destroy happening
  because the channel is being closed, but the Background destroy method even
  delays its own shutdown by a turn of the PBackground event loop for
  MessageChannel life-cycle reasons.

### Part 14:

## Freeze/Suspend Stuff

// Shared workers are only frozen if all of their owning documents are
// frozen. It can happen that mSharedWorkers is empty but this thread has
// not been unregistered yet.

// Shared workers are resumed if any of their owning documents are thawed.
// It can happen that mSharedWorkers is empty but this thread has not been
// unregistered yet.

## Previous review tracking:
- r+  part 1 - Moving SharedWorker in a separate folder
  https://bugzilla.mozilla.org/attachment.cgi?id=8951707&action=edit
- r+  part 2 - PSharedWorker protocol
  https://bugzilla.mozilla.org/show_bug.cgi?id=1438945#a404500_151407
- r+  part 3 - SharedWorkerService and SharedWorkerManager
  https://bugzilla.mozilla.org/show_bug.cgi?id=1438945#c9
  - Comments about suspended != mActors.Length().
    - In the review, the check was `if (suspended != mActors.Length())` and is
      now `if (suspended != 0 && suspended != mActors.Length())`.
      - The previous problem was that the early return would never un-suspend
        beause the early return would happen

  - suggested adding comments
- r+  part 4 - errors and communications
  https://bugzilla.mozilla.org/show_bug.cgi?id=1438945#c10
  - suggested adding comments
- r+  part 5 - console test
  https://bugzilla.mozilla.org/show_bug.cgi?id=1438945#c11
  https://bugzilla.mozilla.org/show_bug.cgi?id=1438945#c38


## Outstanding items to loop:
- Part 3: bfcache suspend/freeze test coverage issue.  See if there's an
  existing test that can be name-checked.

### Documentation that should be added:
Probably want to provide this as a patch to layer on top?
- SharedWOrkerTypes.ipdlh
  - SharedWorkerLoadInfo:
    - URL's.

## Diagrams

###

```

```

## Bitrot Fixing:
All other changes were obvious.

### shared_3.patch

#### deleted hunk transform
    created = true;
  } else {
    // Check whether the secure context state matches.  The current compartment
    // of aCx is the compartment of the SharedWorker constructor that was
    // invoked, which is the compartment of the document that will be hooked up
    // to the worker, so that's what we want to check.
    shouldAttachToWorkerPrivate =
      workerPrivate->IsSecureContext() ==
        JS_GetIsSecureContext(js::GetContextCompartment(aCx));
##### to
    created = true;
  } else {
    // Check whether the secure context state matches.  The current realm
    // of aCx is the realm of the SharedWorker constructor that was invoked,
    // which is the realm of the document that will be hooked up to the worker,
    // so that's what we want to check.
    shouldAttachToWorkerPrivate =
      workerPrivate->IsSecureContext() ==
        JS::GetIsSecureContext(js::GetContextRealm(aCx));

#### error bubble uses enum now
    RefPtr<AsyncEventDispatcher> errorEvent =
      new AsyncEventDispatcher(sharedWorker, NS_LITERAL_STRING("error"), false);
##### to
      new AsyncEventDispatcher(sharedWorker,
                               NS_LITERAL_STRING("error"),
                               CanBubble::eNo);

#### remote_1
