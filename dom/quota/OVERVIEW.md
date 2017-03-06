## Basics ##


### Threads ###
The QuotaManager is a PBackground singleton.  Most of its activities happen on
its "Storage I/O" LazyIdleThread which gets manually shutdown.  Creation does
involve bouncing back and forth between the main thread and background thread.

### Clients ###
* indexedDB
* asmjscache
* cache

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

### Upgrade Logic ###



#### Nitty Gritty ####
Per-method descriptions, ordered in a sort of topological sort because some
methods are called multiple times and I need the columns.

* QuotaManager::EnsureStorageIsInitialized:
  * if database was not at storage v1 (which is the upgrade case from v0):
    * MaybeUpgradeIndexedDBDirectory
    * MaybeUpgradePersistentStorageDirectory
    * MaybeRemoveOldDirectories
    * UpgradeStorageFrom0To1

* QuotaManager::MaybeUpgradeIndexedDBDirectory: (called by
  QuotaManager::EnsureStorageIsInitialized)
  * move profile/indexedDB to profile/storage/persistent

* QuotaManager::MaybeUpgradePersistentStorageDirectory
  * bails if the "profile/storage/persistent" directory no longer exists
  * Uses CreateOrUpgradeDirectoryMetadataHelper::CreateOrUpgradeMetadataFiles
    once on "persistent" (aPersistent=true) and once on "temporary"
    (aPersistent=false)
  * Then moves "persistent" to be "default", which also avoids the routine
    firing again.

* CreateOrUpgradeDirectoryMetadataHelper::CreateOrUpgradeMetadataFiles:
  (called by QuotaManager::MaybeUpgradePersistentStorageDirectory)
  * Operates on either the persistent or temporary directories, and it's fine if
    they don't exist (because there never was any data), happily bails.
  * Enumerates over all files, ignoring OSX DSStore files, complaining but
    ignoring other files, special-casing "moz-safe-about+++home" by deleting it,
    then processing directories.
  * If persistent (== not temporary), invoke MaybeUpgradeOriginDirectory to
    possibly move stuff under idb.  This is done only for persistent because
    this is the second stage of handling MaybeUpgradeIndexedDBDirectory and the
    migration from the version=-2 IndexedDB-only world.  Temporary did not
    exist at that point.
  * AddOriginDirectory gets invoked which appends an OriginProps to
    mOriginProps, initializes its mDirectory, and tries to fill-in the mSpec,
    mAttrs, and mType.  For a chrome directory, those are hard-coded, but
    otherwise it uses PrinipalOriginAttributes::PopulateFromOrigin and
    OriginParser::ParserOrigin to try and extract them from the directory name.
    POA gets first try and is doing the modern thing and looking for the
    "^suffix" (and yes, the ^ is cool with dumb filesystems) and stripping and
    processing.  The remaining originNoSuffix (which is the whole directory
    leaf name if this is pre-v2) gets processed by OriginParser::Parse which
    mutates the spec to be a real spec based on its parsing.  If an app id was
    detected in the parse, a fresh POA with explicit (mAppId,
    mInIsolatedBrowser) is returned, otherwise the original one provided by
    POA's PopulateFromOrigin is propagated.
  * If (!mPersistent), calls GetDirectoryMetadataGetDirectoryMetadata to check
    the .metadata file.  (The !mPersistent check is because the call to
    MaybeUpgradeOriginDirectory above already interpreted the lack of a
    .metadata file as a need to do something and touched it into existence if it
    did not exist.  Similarly, in our DoProcessOriginDirectories method,
    mPersistent again gets a special-case which will 100% invoke
    CreateDirectoryMetadata.)  If it did not exist, GetLastModifiedTime is used
    to set the originProps mTimestamp and mNeedsRestore is flagged.  If it did
    exist and the .metadata file was long enough to include the isApp boolean,
    then mIgnore is set.  (If not set, we try and append the flag into
    existence.)
  * elif !IsOriginWhitelistedForPersistentStorage (and possibly mPersistent,
    which would be the case for something that was in persistent but is soon
    ending up in default), GetLastModifiedTime is used to set the mTimestamp to
    use as the atime.


* CreateOrUpgradeDirectoryMetadataHelper::MaybeUpgradeOriginDirectory:
  (called as part of this->CreateOrUpgradeMetadataFiles)
  * If there's no ".metadata" file in the directory, idempotently create the
    "idb" directory but warning if it already existed.
  * Move everything that's not the idb directory into the idb directory.
    (There's no other skipping since only the .metadata file that we know didn't
    exist could be in there.  The OSX DSStore could get moved if it exists, but
    it's not a big concern.)
  * Create the metadata file as 0644 but don't actually do anything with it.

* CreateOrUpgradeDirectoryMetadataHelper::GetDirectoryMetadata
  (called from this->CreateOrUpgradeMetadataFiles, different from the
   UpgradeDirectoryMetadataFrom1To2Helper's method of the same name.)
  * Opens the .metadata file as a binary stream
  * reads [uint64_t timestamp, Cstring group, Cstring origin, bool isApp whose
    value is ignored but where the file length indicates HasIsApp] and returns.

* GetLastModifiedTime (in microseconds, a la PRTime):
  * Walks a directory and its children, accumulating the most recent file
    modification time it finds for files that aren't QM's own ".metadata" or
    ".metadata-v2" file (both of which track/store atime), or the inescapable
    OSX .DS_Store files created by its Finder.

* CreateOrUpgradeDirectoryMetadataHelper::DoProcessOriginDirectories:
  * Iterates over the originProps.
  * If (mPersistent) (only persistent/temporary existed in the being-upgraded
    scheme), CreateDirectoryMetadata() for sure.  This is important because if
    the file didn't exist, MaybeUpgradeOriginDirectory only touched .metadata,
    it didn't populate it.
    * If IsOriginWhitelistedForPersistentStorage() returns true, move
      the directory to the new "permanent" dir (which we may need to create).
      If the target directory already existed, QM_WARNING and delete the source
      directory, thereby favoring the already-upgraded target.  (This happened
      if the user had been jumping around between Gecko versions.)
  * Elif (not persistent and) we had marked it mNeedsRestore because the
    .metadata file did not exist, create it using what's in originProps.  So the
    last modified time has become our last access time, and all the origin data
    is from the directory name inference we did.
  * Elif !mIgnore, open the .metadata file and append mIsApp.  (mIgnore was set
    if the isApp bool was already present.)

* QuotaManager::MaybeRemoveOldDirectories: (called by
  QuotaManager::EnsureStorageIsInitialized)
  * nukes profile/indexedDB and profile/storage/persistent.

* QuotaManager::UpgradeStorageFrom0To1: (called by
  QuotaManager::EnsureStorageIsInitialized)
  * Iterates over (persistent, temporary, default), and for each, creating a
    UpgradeDirectoryMetadataFrom1To2Helper and invoking its UpgradeMetadataFiles
    method for each dir.
  * Marks the storage version as upgraded.

* UpgradeDirectoryMetadataFrom1To2Helper
  * AddOriginDirectory: see CreateOrUpgradeMetadataFiles

* UpgradeDirectoryMetadataFrom1To2Helper::GetDirectoryMetadata
  (called by this->UpgradeMetadataFiles, differs from
  CreateOrUpgradeDirectoryMetadataHelper version by returning isApp, which makes
  sense given its known to be in a better state.)
  * Opens the .metadata file as a binary stream
  * Reads [uint64_t timestamp, Cstring group, Cstring origin, bool isApp] and
    returns.

* RestoreDirectoryMetadata2Helper::RestoreMetadata2File:
  * AddOriginDirectory: see CreateOrUpgradeMetadataFiles

* UpgradeDirectoryMetadataFrom1To2Helper::DoProcessOriginDirectories
  * Iterates over all the accumulated origin props, and for each:
  * If mNeedsRestore


### Legacy Directories ###
(and how they get upgraded/nuked out of existence)

* "indexedDB" under the mBasePath used to be where IDB stuff lived.
  MaybeUpgradeIndexedDBDirectory tries to move the dir to storage/persistent if
  it does not already exist.  If the IDB dir continues to exist after that (say,
  due to storage/persistent already existing due to moving back and forth
  between Gecko versions around when the migration was introduced), the
  newly introduced MaybeRemoveOldDirectories() will recursively delete it.
* "storage/persistent" under the mBasePath, another step on the way to the
  current storage/{default,permanent,temporary} scheme.  If "storage/default"
  does not already exist (say due to jumping between gecko versions), uses
  CreateOrUpgradeDirectoryMetadataHelper (using aPersistent = true) to migrate
  the directory tree.  Also uses it to migrate the temporary directory
  (using aPersistent = false).  These both process their directories in-place.
  Then persistent gets moved to default. *does the upgrade process move things
  over to permanent as appropriate?*.



## Startup ##

### Creation ###

Creation is a little weird; GetOrCreate takes a runnable to invoke once things
are created.  It's expected to be invoked on the background thread, but it
dispatches a CreateRunnable that starts out on the main thread.  CreateRunnable
then has a state machine where each step may run on different threads.
Currently, this is what happens:
* State::Initial: main thread, Init(): Resolve NS_APP_INDEXEDDB_PARENT_DIR
  ("indexedDBPDir") from the directory service if available, failing over to
  NS_APP_USER_PROFILE_50_DIR if not.  In practice,
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

## Quota Tracking ##

* DecreaseUsageForOrigin:
* UpdateOriginAccessTime:

## Space Reclamation ##

*CollectOriginsForEviction*
