This file currently contains various notes related to the upgrade process that
are now somewhat outdated thanks to various excellent refactorings carried out
by :janv.  I also have some notes in the /reviews/ directory too.

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
