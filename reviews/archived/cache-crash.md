### Crash Reports ###

weather.com is mentioned a fair bit.
* it uses a single serviceworker and a bunch of caches, many of which are
  sw-toolbox based.
* I browsed to the main site, weather, and then news.  news seemed to result in
  a bunch more stuff being installed.
* reaching steady state after that, there's a fair amount of data in there:
  Check it:
```
$ du -csh *
716K	caches.sqlite
32K	caches.sqlite-shm
512K	caches.sqlite-wal
0	context_open.marker
11M	morgue
12M	total
```

bell.ca tv variants:
* Main suspicious idiom is:
  * addEventListener('activate', function(event) {...}
    * event.waitUntil(cache.keys().then(function(cacheNames) {...}
      * return Promise.all(cacheNames.filter().map(function(cacheName) {
        return caches.delete().then(random callback func that can return a
        promise}).
      * The filter is to decide what to delete.  It skips '$$$inactive$$$' and
        the current version, but keeps anything with the CACHE_PREFIX.  This
        avoids deleting caches not managed by the given SW (as indicated by
        prefix.)

#### GDB investigation of non-crashing shutdown triggered by our client reset.

* we survive to the event loop spinning

In my breakpoint on a QM op, I found a frame with a synchronously invoked QM op under QM::UpdateOriginAccessTime...
* message received to CacheStreamControlParent::RecvNoteClosed, chains:
  * StreamList::NoteClosed
  * CacheStreamControlParent::Shutdown
  * PCacheStreamControlParent::Send__delete__
  * PCacheStreamControlParent::DestroySubtree
  * CacheStreamControlParent::ActorDestroy
  * (many frames of the destroy function doing RefPtr stuff)
  * StreamList::~StreamList (many frames)
  * Context::Releaseown a QuotaClient in the hopes of being able to reproduce this in a mochitest.  (nsIQMS's reset() doesn't shut down the QuotaClients.)  I've got the helper working, but it hasn't borne fruit yet.  Presumably because I still need to cause the PBackground channel to self-destruct before triggering the shutdown.
  * Context::~Context (many frames)
  * QM::UpdateOriginAccessTime

NEW WILD THEORY: We get screwed by a previous client's spun event loop!
* I don't think this necessarily affects anything because it's not clear that
  anything interesting occurs during shutdown prior to the call to our
  ShutdownWorkThreads.  QM only shuts down the other clients preceding us,
  which are indexedDB and asmjscache.  They can spin, but because of the lack
  of notable changes, their spin seems no different from if the events had
  happened on the bare event loop before QM shutdown.

OTHER: Inherent unsoundness of race between IPC shutdown ActorDestroy call and
our call of Send__delete__ that explicitly destroys?
 * Variants:
   * A: ActorDestroy hits before we get to the NoteClosed invocation.
   * B: NoteClosed
 * Conclusion: It doesn't seem like the shutdown process supports this scenario.
   Especially since most crashes are in non-e10s which rules out inadvertent
   content process death as a potential source of an arbitrary ActorDestroy
   happening.

Misuse by others: A naming failure results in the wrong actor doing stuff.
  * Nope.  The actors are self-controlled.

#### UAF in Context ###

Use-after-free is another layer up, in Context.  It's got a weak-ref to the
StreamList via the only-weaf-ref-is-possible Context::Activity.
* The entry point is still StreamList::Cancel().
  * StreamList::Cancel still just amounts to issuing a SendCloseAll() down to
    the child which is thread-safely idempotent.
  * Other than the Context::CancelAll() entrypoint, there's
    Context::CancelForCacheId.  This is used consistently as the
    StorageDeletAction complete handling.  If the delete completes and there are
    no outstanding refs, then if the context isn't already canceled,
    Context::CancelForCacheId is invoked and a DeleteOrphanedCacheAction
    dispatched (via the canceled Context's Context::Dispatch (?!)).  If there
    were refs then SetCacheIdOrphanedIfRefed sets mOrphaned true and the logic
    runs when Manager::ReleaseCacheId() is invoked and the ref count has gone to
    zero.
    * DeleteOrphanedCacheAction is a SyncAction, not a BaseAction.  It has no
      listener.  The use of SyncAction seems to be a misnomer; it looks like
      the only real intentional choice was not to use BaseAction (with its
      listener stuff) and/or maybe to be able to avoid duplicating the limited
      logic in RunWithDBOnTarget that dispatches to RunSyncWithDBOnTarget.

MOOT THEORY: The shutdown screws up the Context thread ownership, allowing
MaybeAllowContextToClose to be invoked in such a way that the last owning
reference ends up on the I/O thread.
* The ThreadSafeHandle doesn't seem to be strong enough on its own.
* RefPtr<Context> holders:
  * QuotaInitRunnable, mContext.  Happens too early to care, but idiom is same
    as ActionRunnable.
  * ActionRunnable, mContext.
    * Destructor asserts !mContext
    * Clear() method calls mContext->RemoveActivity(this) then nulls out.
      * Dispatch() calls (asserted on owning thread) if mTarget->Dispatch fails.
      * Context::ActionRunnable::Run() invokes in STATE_COMPLETING which is
        asserted on owning thread.  (With smart comment about the destructor
        potentially firing on any thread it bounced off of.)

MOOT THEORY: It's the StreamList::~StreamList on the stack invoking
ReleaseCacheId with mContext still knowing about the activity.
* NoteClosing is boring.
* DoomTargetData actually keeps the context alive and it's a boring thing.
* So, no.

MOOT THEORY: StreamList::~StreamList happens as part of a preceding cancellation
and the observer array doesn't protect against that even though it should.
* No, the observer array iterator is

SUB THEORY: mPendingActions' removal/clearing can result in unhappy states?
* Note that Context::OnQuotaInit has specializing handling for
  STATE_CONTEXT_CANCELED to invoke CompleteOnInitiatingThread().  State
  immediately transitions to canceled if the passed-in aRv was bad.  Otherwise,
  the element is just removed or the whole array cleared.
  * For MatchAll(), the OnQuotaInit bailing will activate the mStreamList and
    invoke OnOpComplete() passing the mStreamList.  In the other cases where
    a StreamList would have been passed in (match, keys, storagematch), no
    activation occurs and the streamlist isn't passed out and instead the
    refcount should go to zero.
* mPendingActions is nsTArray<PendingAction> where PendingAction is a
  straightforward struct of RefPtr<Action> mAction, and completely unused
  nsCOMPtr<nsIEventTarget> mTarget for some (legacy?) reason.
* So the StreamList will hang, unactivated and with an empty list, until the
  pending action is cleared.  It will have been registered as an activity, so
  Cancel() will be called, but CloseAll() will have nothing to do.

BAD THEORY: Put op involved.
* It doesn't actually involve a StreamList.

BAD THEORY: Variant on the Manager Shutdown case.  In this case, a Manager
RefPtr continues to be held after MaybeAllowContextToClose is able to trigger
a close.
* Investigation
  * CacheOpParent: No, mManager is saved off at the same type the Execute calls
    are made that adds it as a listener.
  * CacheParent: No, mManager is saved off in the constructor at the same time
    AddRefCacheId is added.


ALT THEORY to the below: There's only a single Context operating, its manager
did not transition to Closing, and it has a orphan deletion in flight because
a deleted cache lost its reference (which did not kill the context/manager),
plus streams that may or may not be active.
* For most interesting race cases, this is seeming more likely, because
  interesting races require there to have been a stream control created, but
  the StreamList will only be given a CacheStreamControlParent if
  AutoParentOpResult invokes SerializeReadStream.
  * The orphan purge can still happen, it just has to be for a different cache.
    * Note that we know from previous investigation that StreamList::~StreamList
      is boring in terms of ReleaseCacheId having cascading effects.

NEW THEORY: Orphaned cache helps create scenario with old manager and old
context still in cleanup creating mNextContext edge case.  Especially because
new Context might not be activated yet.
* Theory of happening:
  * These things can hold a Manager to stop it from closing:
    * CacheOpParent: It registers as a listener as a side-effect of Exeute*
      calls and removes itself when the actor goes away.  So these can obviously
      lose their actors since these are just match/put/delete/etc.
    * CacheParent: Does AddRefCacheId for the cache it corresponds to.  The
      cache instance can obviously go out of scope in a proper event-driven
      idiom.
    * StreamList: Does AddRefCacheId in StreamList::Activate, but only does
      ReleaseCacheId in its destructor.  (It also will addref the bodies.)
  * Against the new manager, Manager::ExecuteStorageOp will create a new
    StreamList() (which immediately invokes AddActivity() against the context)
    and SaveListener() the op, which means that we do not expect the old
    context to have a StreamList, but we do expect the new Context to have one.
    (The old streamlist would have precluded the Manager shutting down because
    once it activated it would have done AddRefCacheId )
    * StorageMatchAction is the only one that will hold onto the StreamList.
    * Context::Dispatch will put the action in mPendingActions because mState is
      STATE_CONTEXT_PREINIT.
  * As such, we expect the old Context to only have its database activity in it.
  * Since only CacheOpParent hold onto Manager instance as a listener, and
    CacheParent via a mCacheIdRefs
    possible for a ServiceWorker that's still alive to allow its Manager to
    close itself down.
  * ServiceWorker update, possibly double-update? of some type.  SW deletes
    cache, gets killed, which causes Manager to go into Shutdown.  New SW spins
    up, it wants to do Cache stuff, gets in line.
* Scenario then:
  * old Context should be holding a strong mNextContext ref to the new Context.
    The new Context's Start() will never have been invoked, so its state will be
    STATE_CONTEXT_PREINIT and it will be holding a fresh mData and passed-in
    mInitAction.
  *
    * Scenario involving queued storage op?
      *

  * old Manager's Shutdown() is invoked (first because it's older and it's an
    ordered list of managers):
    * (we will not fast-path out, we got to be an old manager via
       MaybeAllowContextToClose, not shutdown, at least in our theory.)
    * old Context's CancelAll() will be invoked.
      * Iterate over already-canceled things.
      * For DB actions, which we certainly have, it looks boring with
        cancellation absolutely being idempotent:
        * Default Action Cancel() just sets mCanceled which is used by code to
          call IsCanceled().
        * This is what our DeleteOrphanedCacheAction would check at the top of
          DBAction::RunOnTarget, but it clearly already got past the check and
          into the SQLite code.
        * Manager::CachePutAllAction does CancelAllStreamCopying() which is just
          controlling Necko APIs and not us if we're orphaned.  As noted about
          the cancellation barrier for the deletion, this almost certainly means
          that there was no action of this type in the queue.
      * The StreamList, if there is one, has no idempotent fast-bail.  It will
        re-invoke NotifyCloseAll which will re-invoke CloseAllReadStreams which
        will re-invoke CloseStream on each/every
  * new Context's CancelAll() is invoked:
    * mInitAction will be nulled out, destroying the SetupAction
    * mPendingActions will be cleared
    * mActivityList iterates through, invoking Cancel() on all activities.
      *
    * AllowToClose() invokes ThreadsafeHandle::AllowToClose, we're on
      PBackground already, so direct call-through to AllowToCloseOnOwningThread
      * Invokes DoomTargetData which will create a NullAction with
        aDoomData=true which should get dispatched to its IO thread.
      * Drops mStrongRef.  This should leave the runnable and mNextContext
        holding strong refs to us.

Confusing/illuminating:
* It's ReleaseCacheId that allows the DeleteOrphanedCacheAction to run free.
  This is invoked in 2 places:
  * In CacheParent::ActorDestroy, which should happen as a result of
    CacheParent::RecvTeardown invoking Send__delete__ or because of tree
    shutdown.
    * CacheChild::SendTeardown is called by CacheChild::StartDestroy() which is
      invoked by CacheChild::StartDestroyFromListener() from Cache::~Cache().
      There can be a delay that occurs if child actors are alive.
    * In StreamList::~StreamList().  This is predicated on !!mActivated which is
      strongly paired with StreamList::Activate() where the AddRefCacheId call
      is made.

NEW THEORY: Some madness like how Conext::~Context can end up triggering
QuotaManager::UpdateOriginAccessTime, the Context destructor

Q: What about orphaned bodies?  Could be related and explain why tests so far
failed to reproduce.
* ReleaseBodyId is analog to ReleaseCacheId.

Q: Manager::Release{Cache,Body}Id in the !context case mentions orphaned flag
and RemoveContext().  What does that mean?
* RemoveContext walks both mBodyIdRefs and mCacheIdRefs and invokes
  NoteOrphanedData on the Context so the next open handles things.
* RemoveContext is invoked by Context::~Context.  The mOrphanedData flag is set
  by the NoteOrphanedData call as an expected side-effect.  If there is NO
  orphaned data, the marker file will be deleted to signify clean shutdown.

Q: Could this be a function of some type of privacy setting or preference that
   causes quota manager to nuke storage at shutdown and trigger per-origin
   aborts, exercising the QuotaClient::AbortOperations call that propagates into
   Manager::Abort() that invokes NoteClosing() and then context->CancelAll();
  * We've got nsIQuotaManager.clearStoragesForPrincipal that generates a
    ClearOriginParams request that becomes a ClearOriginOp that will break
    locks.
    * Doesn't seem it, existing users seem to be:
      * pageinfo/permissions.js has a clear IDB that uses it.
      * preferences/SiteDataManager.jsm's remove(hosts)/removeAll() uses it.
        removeAll() at least seems to only be used from gPrivacyPane,
        gAdvancedPane, siteDataSettings.  Some of those seem moot?
      * extensions/Extension.jsm clears on uninstall.
      * ForgetAboutSite.jsm uses it.
  * We've also got a "clear-origin-attributes-data" observer that provides a
    pattern.
    * No, this is for the ContextualIdentityService and BackgroundPageThumbs.
  * No obvious sign of rogue extensions that would be doing some kind of privacy
    thing like that.


Q: Context::ThreadsafeHandle
* Has both a strong and weak Context reference.
* AllowToClose() triggers nulling of mStrongRef on PBackground
  * Callers:
    * Manager::MaybeAllowContextToClose()
      * Manager::RemoveListener
      * Manager::ReleaseCacheId
      * Manager::ReleaseBodyId
  * AllowToCloseOnOwningThread is what actually drops mStrongRef.  If not on the
    owning thread (and therefore on the IO thread, barring some mainthread
    weirdness), AllowToClose dispatches runnable to the PBackground thread.
  * If mStrongRef wasn't already dropped, DoomTargetData is performed which
    dispatches a runnable to the IO thread and then the mStrongRef is dropped.
* InvalidateAndAllowToClose()
  * Similar to AllowToClose, InvalidateAndAllowToCloseOnOwningThread does
    the actual work on PBackground via thread-check and dispatch if needed.
  * But doesn't seem to actually be used?
* The destructor makes sure to release the strong ref on the background thread
  via NS_ProxyRelease.
*

Q: Context::mNextContext
* The references themselves seem sound given that Context::~Context seems
  inductively safe to be invoked only on PBackground.
* Comes into life when there's still an old manager hanging around.  In that
  case the Manager will be hanging around with the Closing state and will be
  passed to Manager::Init which will tell Context::Create about it.
  * Closing state comes from NoteClosing() via one of:
    * Manager::Abort(): As covered elsewhere, this implies explicit privacy
      clearing of the origin.  Shouldn't be this.
    * Manager::Shutdown(): Sets mShuttingDown, does NoteClosing(), and does
      Context::CancelAll().
    * Manager::MaybeAllowContextToClose(): If there are no Listeners (added when
      a parent first issues a request, so they may never add themselves, and
      removed when the actor dies), and no cache or body refs, NoteClosing() and
      let the context close.
      * This can happen if ReleaseCacheId or ReleaseBodyId happened.

A: Context::Data's destructor asserts that it wants to only be released on the
target thread, is that true?
  * It's good, see cache/OVERVIEW.md.

A: Manager::~Manager dispatches its ioThread shutdown to the main thread, which
can spin an event loop... any potential fallout from that? (it does allow for
multiple levels of nesting if we expect a bunch of managers to shut down at
once...)
* Crash stacks don't show any sign of the main thread joining on any IO threads.
  Most stacks are of an idle nested event loop, although there's one in the
  cycle collector.
* Plus, ActionRunnable seems to have very sound clearing happening.
* So it seems like no.

Notable code behaviors:
* Manager::BaseAction : SyncDBAction's CompleteOnInitiatingThread only invokes
  Complete() if there's a Listener, as retrieved from Manager::GetListener with
  the initially passed in ListenerId.  Could there be a race with bad cleanup
  happening?

untraced scenario thoughts:
* supply side: create cache, do an add that we intentionally hang, delete cache,
  orphaning it.
* retrieve side: create cache, have thing in it, do a match, hold onto the
  result, delete cache, orphaning it.

#### Crash Stack
Preceding, per analysis:


crash stack (down, deeper):
* CacheQuotaClient::ShutdownWorkThreads
(missing frames. inlined?)
* Manager::ShutdownAll
* Factory::ShutdownAll()
  * iterates over mManagerList, invoking Manager->Shutdown on each.
  * not in our crash signature, but it will spin the event loop after that's
    invoked.
(end missing frames)
* Manager::Shutdown
  * invokes CancelAll() on its Context if it has one.
* Context::CancelAll()
  * in the mActivityList iterator, pulling out Activity*'s and invoking Cancel()
    on each.
  * StreamList subclasses Context::Activity, hence its Cancel() method.
* StreamList::Cancel
  * StreamList is refcounted.
* StreamList::CloseAll() (gets inlined), calls
* CacheStreamControlParent::CloseAll() **transition into dead class**, but no
  accesses occur to cause an explosion because the calls don't have to do any
  virtual dispatch or field accesses.
  * CacheStreamControlParent is IPC-managed, no refcounting.
  * Subclasses StreamControl, which is where we crash.
* CacheStreamControlParent::NotifyCloseAll()
* StreamControl::CloseAllReadStreams() **CRASHES HERE** iterating over
  mReadStreamList.

So there's an ActorDestroy asymmetry.

#### More full understanding of shutdown process

If we had continued, upon completing CSCP::NotifyCloseAll, we would have done:
* PCacheStreamControlParent::SendCloseAll =>
IN CHILD:
* CacheStreamControlChild::RecvCloseAll
  * StreamControl::CloseAllReadStreams()
    * ReadStream::Controllable::CloseStream IS ReadStream::Inner::CloseStream
      * ReadStream::Inner::Close
        * ReadStream::Inner::NoteClosed
          * state guard
          * if on mOwningThread, immediately invokes
            ReadStream::Inner::NoteClosedOnOwningThread, otherwise involves a
            NoteClosedRunnable instance which defers to that.
            * ReadStream::Inner::NoteClosedOnOwningThread
              * StreamControl::NoteClosed
                * StreamControl::ForgetReadStream
                * CacheStreamControlChild::NoteClosedAfterForget
                  * PCacheStreamControl::SendNoteClosed

IN PARENT:
* CacheStreamControlParent::RecvNoteClosed
  * StreamList::NoteClosed
    * only iff mList.IsEmpty() && mStreamControl do we continue to...
    * CacheStreamControlParent::Shutdown
      * Send__delete__(this)
        * PCacheStreamControlParent::DestroySubtree
          * CacheStreamControlParent::ActorDestroy

##### Worker wrinkle; it's fine.

CacheStreamControlChild::StartDestroy ends up getting notified in the event the
CacheWorkerHolder (a WorkerHolder subclass that's necessary because of IPC
being its own ownership space) gets Notify()ed or is added to an already
Notify()ed holder.  It will invoke RecvCloseAll() like if the parent had told
it to shutdown.  This may optionally be delayed until NoteClosedAfterForget
has been invoked for all the read streams.

This is fairly safe.  It's just a question of who triggers the close.  The
Close gets passed through to ReadStream::Inner::NoteClosed() which fast-bails
if already closed.  And if it has to dispatch to another thread, that thread
does a compareExchange and bails.  So there ends up being no race.


#### ActorDestroy investigation

CacheStreamControlParent::ActorDestroy:
* bails out fast if !mStreamList, suggesting if it gets confused about
  its mStreamList, we could end up here.
* if it didn't bail, it calls StreamList::RemoveStreamControl(this)
  which should null out its mStreamControl.
* Then it nulls out its mStreamList

The ownership cycle is established by StreamList::SetStreamControl; it saves the
value and then invokes CacheStreamControlParent::SetStreamList.

The obvious family of gotchas (and our investigation results):
* Async initialization leads to StreamList::SetStreamControl being invoked after
  the actor has been destroyed.
  * This seems impossible.  It's literally created then set, and we're already
    only assigning if it makes it through the constructor.  If it got destroyed,
    it's not getting saved to mStreamControl.
* Ownership multiplicity invariant is violated.
  * The StreamControl is explicitly created for the single aStreamList, and
    although it may be set to the list multiple times, the redundant set is
    expected and has assertion guards (on both sides).
  * Per the OpComplete invocations, the ones passing an mStreamList null it out
    immediately following the call, so the invariant can't be violated.
* Confusion in CacheStreamControlParent leading to null mStreamList after setup,
  resulting in the StreamList never hearing when it dies.
  * No, not in a straightforward way.  It only gets nulled out in the one place.
  * What about in the multiple SetStreamControl case?  Could the ActorDestroy
    happen between loop invocations?
    * No, the Add calls happen in a tight loop.
* StreamList invoked StreamControl's Shutdown() method (which does
  Send__delete__, thereby invoking ActorDestroy) but a StreamList reference is
  still hanging around somehow.
  * There's no case where the Shtudown calls magically hold onto a reference.
    This again ends up needing to be a case of multiple ownership.



Our static backtrace of the ownership being set is:
* AutoParentOpResult::SerializeReadStream
  * If there's no mStreamControl yet, one is created via newing a
    CacheStreamControlParent, and constructing it.
  * aStreamList->SetStreamControl(mStreamControl) is unconditionally called.
    This is expected to be called multiple times in some cases.
* ::Add(SavedRequest) and ::Add(SavedResponse) via
  ::SerializeResponseBody() invoke it.
* CacheOpParent::OnOpComplete(...aStreamList)
  * Creates the AutoParentOpResult in here, and then does its result.Add() calls
    in a list, always with the same aStreamList.
* Manager::WILDAction::Complete
  * Okay, there are a ton of these, but the hygiene seems good; mStreamList gets
    nulled out following the calls to OnOpComplete.  Auditing
    * CacheMatchAction, CacheMatchAllAction, CacheKeysAction,
      StorageMatchAction, : good mStreamList nulled
    * CacheDeleteAction: StorageHasAction, StorageOpenAction,
      StorageDeleteAction, StorageKeysAction, ExceuteCacheOp/ExecuteStorageOp/
      ExecutePutAll closing variants,
      CachePutAllResult::CompleteOnInitiatingThread: no mStreamList in the first
      place.

### Understand when various things are shutting down at quitting time ###

Does content shutdown before QM?  Sorta:
* ContentParent shuts down child processes at "profile-before-change", which is
  before QuotaManager shutdown occurs.

The things that happen prior to QM's notification are:
* "profile-change-net-teardown"
* "profile-change-teardown"
* A JS_GC of dom::danger::GetJSContext() is triggered.  (Is that chrome?)
* "profile-before-change"

Do we have obvious shutdown observers happening?
  * dom/base: Not really.
    * nsContentUtils has RegisterShutdownObserver, but that's "xpcom-shutdown"
    * nsJSEnvironment.cpp clears timers on "xpcom-shutdown"
  * docshell/: No.  Only TimelineConsumers does, and it just cares about
    removing observers.


Order of shutdown of relevant things:
* QuotaManager shuts down.  This includes spinning an event loop on the main
  thread to wait for shutdown on PBcakground.
* RuntimeService (and ServiceWorkerManager) shut down, terminating the workers.
* PBackground shutdown is triggered, including a failsafe that tries to kill
  everything.

More details about the order:
* "profile-before-change"
  * ContentParent triggers ShutDownProcess(SEND_SHUTDOWN_MESSAGE) and spins
    until !mIPCOpen or the process if forcibly terminated.
    * ContentChild::RecvShutdown notifies "content-child-shutdown", but that's
      largely boring; it does some observer cleanup for ContentObservers and
      lets Telemetry report.  Then calls ::SendFinishShutdown()
    * ContentParent::RecvFinishShutdown invokes ShutDownProcess(CLOSE_CHANNEL):
      * QuotaManagerService::AbortOperationsForProcess() gets invoked, but
        our QuotaClient doesn't care and all that QM does is notify the clients.
      * IToplevelProtocol::Close().  (But is this only for PContent?)
      * MarkAsDead() removes the contentparent from sBrowserContentParents.
    * ContentParent::ActorDestroy is where mIPCOpen gets cleared.


  * ServiceWorkerRegistrar does PBackground::SendShutdownServiceWorkerRegistrar
    and then spins until it completes.

* "profile-before-change-qm": (AKA PROFILE_BEFORE_CHANGE_QM_OBSERVER_ID)
  * triggers QM shutdown; dispatches runnable to PBackground thread, then spins
    until it completes.
  * dispatched by nsXREDirProvider::DoShutdown(), invoked by
    ScopedXPCOMStartup::~ScopedXPCOMStartup
    * XREMain::XRE_mainInit initializes and sends out of scope.
* "xpcom-shutdown"
  * triggers RuntimeService and ServiceWorkerManager shutdowns, no explicit
    ordering.
    * ServiceWorkerManager::MaybeStartShutdown() kills all update timers,
      job queues.  I think it's leaving worker shutdown to the
    * RuntimeService::Shutdown issues Kill() call to all top-level workers.
      This does include ServiceWorkers.
  * dispatched by ShutdownXPCOM
    * ScopedXPCOMStart::~ScopedXPCOMStartup invokes after DoShutdown.
* "xpcom-shutdown-threads": ParentImpl::ShutdownObserver::Observe
  * triggers PBackground shutdown, also RuntimeService Cleanup()
  * dispatched by ShutdownXPCOM
    * ScopedXPCOMStart::~ScopedXPCOMStartup invokes after DoShutdown.

Note that on Windows the shutdown process goes from "quit-application-granted"
straight through "profile-before-change-qm" and then it explicitly exits via
`_exit(0);` and it says it doesn't bother to close the window.  This does imply,
however, that maybe the user quit by closing the window, in which case we would
expect a content race.
