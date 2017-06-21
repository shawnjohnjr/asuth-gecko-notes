### Runnables ###

QuotaObject:
* StoragePressureRunnable

QuotaManager:
* CreateRunnable
* ShutdownRunnable

anonymous:
* CollectOriginsHelper
* StorageDirectoryHelper

OriginOperationBase
* FinalizeOriginEvictionOp
* NormalOriginOperationBase
  * SaveOriginAccessTimeOp: triggered by QuotaManager::UpdateOriginAccessTime,
    which in turn is called by RegisterDirectoryLock when the first lock for an
    origin is registered, and by UnregisterDirectoryLock when the last lock is
    unregistered.
  * QuotaUsageRequestBase via PQuotaUsageRequest using UsageRequestParams
    * GetUsageOp: from AllUsageParams.
    * GetOriginUsageOp: from OriginUsageParams.
  * QuotaRequestBase: via PQuotaRequest using RequestParams
    * InitOp: from InitParams.  Issued by QMS::Init()
    * InitOriginOp: from InitOriginParams.  Issued by
    * ResetOrClearOp: from either ClearAllParams (aClear=true) or ResetAllParams
      (aClear=false).
    * ClearOriginOp: from ClearOriginParams
    * ClearDataOp: from ClearDataParams
    * PersistRequestBase:
      * PersistedOp: from PersistedParams.  Triggered by thbubblese
        StorageManager.persist() storage API request call.
      * PersistOp: from PersistParams.  Triggered by the
        StorageManager.persisted() storage API query call.

NormaOriginOperationBase and aExclusive/mExclusive.  Only the following ops set
it.  And they do it because they're testing or explicit clears.
* ResetOrClearOp.  Because it needs to break everyone else's existing locks.
* ClearOriginOp
* ClearDataOp
see also:
* QuotaManager::CreateDirectoryLockForEviction creates an exclusive
  DirectoryLockImpl

#### OriginOperationBase states ####
There's a neat state machine here.

* State_Initial, calls (its private) Init() on PBackground thread:
  * Advances to Initializing on entry.  Then, if mNeedsMainThreadInit,
    re-dispatches itself to the main thread to process the Initializing state.
    If not, advances state again to FinishingInit and invokes Run() to
    immediately process that state.
    * mNeedMainThreadInit gets set in the subclass's Init() method which,
      although sharing the same name, does not hide our method or steal the call
      or anything.  (Its own Init is private and non-virtual.)
* State_Initializing (on main thread, may be skipped), calls InitOnMainThread()
  * calls DoInitOnMainThread() and advances to FinishInit, re-dispatching to
    the PBackground ("owning") thread.
  * This is where subclasses do PrincipalInfoToPrincipal and
    QuotaManager::GetInfoFromPrincipal which need to be main-thread only because
    Principals are main-thread only.
* State_FinishingInit, calls FinishInit() on PBackground thread:
  * Fast fails if QM::IsShuttingDown(), advances state
    (to CreatingQuotaManager).  Then if mNeedsQuotaManagerInit and it's not
    there yet (QuotaManager::Get() returns null),
    QuotaManager::GetOrCreate(this) is invoked.  That will eventually result in
    the op's Run() method being invoked and causing the next state to execute,
    which calls open.  Or if the QM already existed or wasn't needed, we
    directly call Open() ourselves.  (Open advances state internally.)
* State_CreatingQuotaManager, calls QuotaManagerOpen() on PBackground thread:
  * This calls QuotaManager::Get() to sanity check it's available, then calls
    Open() which in the NormalOriginOperationBase advances state and then calls
    QM::Get()->OpenDirectoryInternal.
* State_DirectoryOpenPending, calls DirectoryOpen() on PBackground thread:
* State_DirectoryWorkOpen, calls DirectoryWork() on QM's single IO thread:
  * if mNeedsQuotaManagerInit, invokes EnsureStorageIsInitialized().  This
    checks the PROFILE/storage.sqlite version and does upgrades if it didn't
    exist or is otherwise not up-to-date.
* State_UnblockingOpen, calls UnblockOpen() on PBackground thread
  * under NormalOriginOperationBase, invokes SendResults(), nulls out
    mDirectoryLock, and calls AdvanceState, advancing to State_Complete and
    never rescheduling itself.
* State_Complete, never gets run.
* failures result in a call to Finish(rv)

#### Testing Operations ####
