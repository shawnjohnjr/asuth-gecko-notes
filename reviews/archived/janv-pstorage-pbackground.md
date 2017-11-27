# Move PStorage to PBackground #
https://bugzilla.mozilla.org/show_bug.cgi?id=1350637

## Patch Notes ##

### Part 1: Move PStorage stubs from PContent to PBackground ###

#### Minutae Summary ####
* ContentChild.{h,cpp}: Remove Alloc/Dealloc
* ContentParent.{h,cpp}: Remove Alloc/Dealloc
* PContent.ipdl: Remove PStorage refs, including nested(inside_cpow) constructor
* dom/ipc/moz.build: removed /dom/storage include, the PBackground stuff is in
  ipc/glue, which is what gained the import.
* LocalStorageCache.cpp:
  * FIXME in LocalStorageCache.cpp LocalStorageCache::StartDatabase()
    * This changes where the StorageDBThread is spun up from, which makes sense.
    * Addressed in part 2 with the complete removal in favor of consistent
      PBackground idiom where child is always used.
* PStorage.ipdl => PBackgroundStorage.ipdl:
  * Protocol renamed to be PBackgroundStorage, which does nicely disambiguate
    what's going on if people are talking about PStorage, tec.
  * Preload loses its nested(inside_cpow) which makes sense since we don't have
    CPOW's on PBackground.
* StorageIPC.cpp:
  * Adds Background header imports.
  * StorageDBChild::Init creates PBackground on demand, changes actor creation.
  * *FIXME in StorageIPC.cpp*
    * The diskSpaceWatcher logic (that's now moot without b2g) got #if 0'd.
  * Alloc/RecvConstructor/Dealloc implemented.
    * Dealloc uses ReleaseIPDLReference suggesting new scheme.
* StorageIPC.h
  * Header imports changed for its own specific PBackgroundStorage* subclasses.
  * Alloc/Recv/Dealloc prototypes.
* dom/storage/moz.build: ipdl filename changed appropriately.
* BackgroundChildImpl.cpp:
  * added StorageIPC.h import as needed
  * using of StorageDBChild namespace.
  * Alloc/dealloc implemented.  Alloc does crash because the create should be
    manual, dealloc does ReleaseIPDLReference.
* BackgroundChildImpl.h: corresponding Alloc/Dealloc prototypes.
* BackgroundParentImpl.cpp:
  * StorageIPC.h added
  * Alloc/Recv/Dealloc with helper callouts and parent background thread
    assertions.
* BackgroundParentImpl.h: protos added.
* PBackground.ipdl
  * PBackgroundStorage include added
  * PBackgroundStorage manages added
  * async PBackgroundStorage constructor sans cpow stuff handled.
* ipc/glue/moz.build: added the /dom/storage include, offsetting the
  dom/ipc/moz.build removal.
* ipc/ipdl/sync-messages.ini: The sync message whitelist of PStorage::Preload
  was updated to compensate for the rename to PBackgroundStorage::Preload

### Part 2: Core changes for LocalStorage on PBackground ###

* LocalStorageCache.{h,cpp}
  * Lost `StorageDBBridge* sDatabase`, `bool sDatabaseDown`
  * LocalStorageCache::StartDatabase, GetDatabase, StopDatabase all removed.
    * All references to sDatabase replaced with StorageDBChild::Get().
    * Unguarded, but commented as safe from existing code, possibly-null call to
      StorageDBChild::Get()->SyncPreload(this) in
      LocalStorageCache::WaitForPreload.
    * This all makes sense because PBackground works in the parent process as
      well, so everything goes through the child rather than directly talking to
      the StorageDBThread sometimes and the StorageDBChild sometimes.
  * StorageUsageBridge gained a VIRTUAL for its threadsafe refcounting, which
    is appropriate given it's an abstract base class.
* LocalStorageManager:
  s/LocalStorageCache::GetDatabase/StorageDBChild::GetOrCreate/
* StorageDBThread.cpp
  * *FIXME constructor*
  * InitHelper runnable
  * NoteBackgroundThreadRunnable: Used by Init() to let StorageObserver save off
    a reference to the BackgroundThread.
  * StorageDBThread::GetOrCreate: news and saves off as sStorageThread from
    background thread.
  * StorageDBThread::Init
    * Asserts background thread, creates InitHelper to synchronously get the
      profile directory and init mozStorageService.
    * Dispatches NoteBackgroundThreadRunnable to main thread for StorageObserver
  * ShutdownRunnable: Triggered by StorageObserver observing
    "profile-before-change" on the main thread, gets dispatched to the
    background thread to invoke StorageDBThread::Shutdown and null out
    sStorageThread, then bounce itself back to the main thread to set the
    bool& mDone so the main thread event loop spinning can terminate.
* StorageDBThread.h
  * FIXME above StorageDBBridge about refactoring it into nothing; not addressed
    in stack, I've requested a follow-up bug to be filed and referenced from the
    code.
  * StorageDBThread ceased to subclass StorageDBBridge.
  * GetOriginsHavingData removed from StorageDBBridge, with the StorageDBThread
    signature becoming non-virtual and gaining a comment.
* StorageIPC.cpp:
  * StorageDBChild::GetOrCreate:
    * follows creation idiom used by StorageDBThread.
    * *comment about invoking LocalStorageManager::Ensure()* which says
      DOMSessionStorageManager may invoke.  Need to double-check that.
  * ShutdownObserver:
    * created, added as an observer.
    * removes self at xpcom-shutdown, releases sStorageChild, nulls it out,
      sets shutdown flag per idiom.
  * LoadRunnable::Run() now nulls out mParent in the Run().  This is desirable
    given that mParent is a StorageDBParent and Run() happens on the background
    thread, whereas letting the destructor handle it could result in the
    refcount hitting zero on the I/O thread.  That mainly matters to
    ObserverSink->Stop.
  * various custom ::Release implementations that didn't exist before and didn't
    need to exist before because they were virtually inherited with virtual
    destructors.
* StorageIPC.h
  * StorageDBChild no longer subclasses StorageDBBridge which is doomed.
  * some methods gained MOZ_CRASH that per the FIXMEin StorageDBthread.h simply
    just should no longer exist, like AsyncFlush().
* StorageObserver.cpp
  * profile-before-change made parent-process only, xpcom-shutdown removed.
    * This is part of changing the existing || for both to just
      profile-before-change.
    * Previously invoked LocalStorageCache::StopDatabase which was process-aware
      but has now been removed with on-demand StorageDBChild::Get() calls.
    * From a correctness perspective, we really do only need to trigger the
      shutdown in the parent given that the db thread does need to be flushed,
      but that's the only place that state gets "stuck" and we don't need or
      really want to add additional life-cycle issues related to shutdown
      otherwise.
  * NoteBackground thread to save off mBackgroundThread from StorageDBThread's
    NoteBackgroundThreadRunnable.
  * a bunch of "fix me"s added that get cleaned up in part 5.


### Part 3: Move mozilla::dom::Optional serialization helper to ipc/glue/IPCMessageUtils.h to make it available to other consumers ###

This part is billm.  I looked into it to understand, suggested removing the XXX
bit.

### Part 4: Implement serialization for mozilla::OriginAttributesPattern, so we can use it on the receiver side of IPC without bouncing to the main thread ###

### Part 5: Factor out the parent actor observer sink to work on the PBackground thread, fix the rest of observer handling to use IPC ###

* StorageIPC.cpp
  * Assertion for StorageDBChild::RecvObserve only happens in non-parent process
    * This is made true by StorageDBParent::Init conditionally only creating and
      registering an oserver sink if IsOtherProcessActor(actor).
    * This is extra desirable because StorageObserver is invoked in both the
      parent and child processes, which is a teeny bit weird.  Although
      RecvObserve only calls Notify, not Observe.
  * StorageDBParent::Init introduced and initially invoked from a runnable,
    although that gets moved in a subsequent patch.
  * StorageDBParent::RecvStartup() exists to invoke StorageDBthread::GetOrCreate
    to trigger creation as a side-effect.
* StorageObserver.cpp
  * delayed database start converted to use new Startup IPC method
  * "cookie-changed" uses somewhat weird combination of both
    StorageDBChild::AsyncClearAll (which just clears mOriginsHavingData if it's
    present), and SendClearAll which only clears the database.
    * However, origins having data is just an optimization to trigger preloads
      which we generally expect to reach a steady state, so it's not a biggie.

### Part 6: Fix a deadlock when main process storage child actor triggers storage thread initialization ###



### Part 7: Convert asynchronous StorageDBParent initialization to be synchronous and fix low disk space checking ###

### Part 8: Implement BackgroundParent::GetLiveActorArray ###

### Part 9: Move Local Storage event broadcasting from PContent to PBackground ###
