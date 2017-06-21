## Basics ##


### Threads ###
The QuotaManager is a PBackground singleton.  Most of its activities happen on
its "Storage I/O" LazyIdleThread which gets manually shutdown.  Creation does
involve bouncing back and forth between the main thread and background thread.

### Clients ###
* indexedDB
* asmjscache
* cache

### Participants ###

* QuotaManager: The PBackground-resident core.  Defined primarily in
  ActorsParent.cpp.
* QuotaManagerService: Main-thread singleton that exists to expose
  nsIQuotaManagerService to XPCOM and receive observer notifications (clear
  origin, idle/non-idle), communicating these requests to/from the QuotaManager
  via PBackground-managed PQuota.
* StorageManager: The StorageManager interface WebIDL binding exposed as
  navigator.storage.

### How Usage is Tracked ###

*GroupInfo*, *OriginInfo*



## Directories / File System Mapping ##

The root directory nets out to be "storage" (mStoragePath) under the profile
directory (mBasePath).  (B2G allowed the parent dir to be elsewhere, but that's
clearly moot.)

* Persistence type directories: "permanent", "default", or "temporary"
* Origin w/suffix directories: The origin (including scheme, domain, and port
  if not default) with ":" and "/" both re-mapped to "+".
  * ".metadata"
  * ".metadata2"
  * "asmjs": asm.js cache, managed by dom/asmjscache.
  * "cache": Service Worker Cache API storage, managed by dom/cache code.
  * "idb": IndexedDB storage, managed by dom/indexedDB code


### Quota Manager On-Disk Data ###

Total Ordering based on the "storage" numbers:
* Version -2: Only IndexedDB existed, everything lived under profile/indexedDB,
  with the IDB files sitting directly in the origin directories.
* Version -1: "storage/{persistent,temporary}", .metadata files existed but may
  not have included the isApp boolean on the end.  (Actually, maybe the isApp
  thing happened at another time.  Doesn't really matter.)
* Version 0: "storage/{permanent,default,temporary}" w/.metadata files
* Version 1:

"Metadata":
* Version 1: .metadata: [uint64_t timestamp, Cstring group, Cstring origin,
  bool isApp] where group and origin have what is now referred to as the
  "jar prefix" crammed on the front.  Note that modern QM will still create
  these files currently and goes out of its way to drop the suffixes.
* Version 2: .metadata-v2: [(u)int64_t timestamp, bool false (reserved for
  navigator.persist()), reserved data 1: 32bit 0, reserved data 2: 32bit 0,
  Cstring suffix, Cstring group, Cstring origin, bool isApp].  Both the origin
  and the group include the suffix.  I think the separate storage of the suffix
  is an evolutionary glitch; initially the origin was retrieved as
  OriginNoSuffix but that changed in bug 1195930 at some point.

"Storage":
* Version 0: (anything before version 1)
* Version 1:


## Startup ##

In short:
* QuotaManager lazy initializes on a few levels.
* The first call to EnsureOriginIsInitialized will trigger a full traversal of
  the storage/default/ and storage/temporary/ directories.

In long:
* QuotaManager::GetOrCreate brings it into existence.  2 sources:
  1. A PQuota request is translated into an OriginOperationBase op that will
     run and invoke QM::GetOrCreate.  (The `Quota` class is the PQuotaParent
     impl, Quotachild is the PQuotaChild impl.)
  2. A QM Client which ends up duplicating the OriginOperationBase state
     machine (either with its own, or the dom/quota/shared/ClientContext.h
     implementation extracted from dom/cache/) does it.
* EnsureStorageIsInitialized checks PROFILE/storage.sqlite, performing
  upgrades if needed.
  * This may be directly triggered by OriginOperationBase when
    mNeedsQuotaManagerInit, or as a side-effect of calls to
    QuotaManager::EnsureOriginIsInitialized.
* EnsureOriginIsInitialized
  * mTemporaryStorageInitialized logic; single-shot.  Calls
    InitializeRepository() for temporary and default, order varying by caller.
    * InitializeRepository walks their mangled origin child directories, calling
      InitializeOrigin for each subdirectory.
    * InitializeOrigin gets invoked for each origin dir, walking its files, and
      for client-named directories (as determined by Client::TypeFromText), it
      invokes the client's InitOrigin() method.
      * As long as the persistence type isn't PERSISTENCE_TYPE_PERSISTENT (which
        is for chrome-privileged stuff; StorageManager.persist() is tracked as
        persisted and doesn't affect this), trackQuota is true and a UsageInfo
        is constructed, passed to the client InitOrigin calls, and then passed
        to InitQuotaForOrigin.
      * InitQuotaForOrigin stores stuff in mGroupInfoPairs (for the group parts)
        and inside its GroupInfo value (for the origin bits).

### Creation ###

Creation is a little weird; GetOrCreate takes a runnable to invoke once things
are created.  It's expected to be invoked on the background thread, but it
dispatches a CreateRunnable that starts out on the main thread.  CreateRunnable
then has a state machine where each step may run on different threads.
Currently, this is what happens:
* State::Initial: main thread, Init(): Resolve NS_APP_INDEXEDDB_PARENT_DIR
  ("indexedDBPDir") from the directory service if available, failing over to
  NS_APP_USER_PROFILE_50_DIR if not.  In practice, the profile dir is used, with
  the parent dir having been a b2g thing (IIRC).
* State::CreatingManager: background/owning thread, CreateManager(): news
  QuotaManager() using that path.
* State::RegisteringObserver: main thread, RegisterObserver():
  * Uses the C++ preferences API from Preferences.h to get live-updated pref
    values for "dom.quotaManager.temporaryStorage.fixedLimit",
    "dom.quotaManager.temporaryStorage.chunkSize", and
    "dom.quotaManager.testing".  (This eliminates the historical need to
    register observers on the branch.)
  * Register a profile-before-change observer for shutdown (this will change
    to profile-before-change2 at some point.)
  * Get a ref to mozIStorageService (we require this for now because of pref
    reasons)
  * Invoke QuotaManagerService::GetOrCreate to expose the XPCOM interface to the
    QuotaManager and tell it about our manager.  (Which is a bit weird in that)
    ::NoteLiveManager/::NoteFinishedManager just set/clear mBackgroundThread.
    When present, runnables are dispatched to the PBackground thread where they
    use QuotaManager::Get.)
* State::CallingCallbacks: background/owning thread, CallCallbacks():
  * Sets gInstance
  * Iterates over the callbacks directly running them.  (Multiple callbacks can
    stack up because there's only ever one gCreateRunnable and the callbacks
    keep getting tacked on.)

### Storage Initialization via EnsureStorageIsInitialized() ###

OriginOperationBase::DirectoryWork calls this when mNeedsQuotaManagerInit is
true.  It checks PROFILE/storage.sqlite for existence and the correct schema
version.  If the file was missing or the wrong version, an upgrade sweep occurs
which potentially transforms the contents of the PROFILE/storage/ directory
every time.

## Quota Tracking ##

* DecreaseUsageForOrigin:
* UpdateOriginAccessTime:

## Space Reclamation ##

*CollectOriginsForEviction*
