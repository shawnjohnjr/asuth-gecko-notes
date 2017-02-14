## Structure Cloning, Persistence, and IPC

Basics:
- The structured clone writes happen in the content process.
- The reads happen in the content process too!
- All the database reads and writes happen in the parent!

Fallout:
- This creates some complexity for the IDB logic in the parent process when
  fetching records because it needs to do things like create blob actors without
  the benefit of being able to deserialize the structured clone data.
  - This is addressed by storing metadata in a separate column next to the
    serialized structured clone.  (Note: the structured clone itself may end up
    being stored externally.)
  - The metadata is an encoded list of numeric file id's.
    - The numeric part is the name of the file on disk under the database's
      ".files"-suffixed dir.
    - The prefix encodes the type of StructuredCloneFile::FileType it is.
      (blob, mutable file, spilled huge structured clone, wasm bytecode, wasm
      compiled).
  - Some metadata is deferred.  Specifically, for Blobs/Files the size/date is
    stored in the structured clone and not stored as metadata in the database
    or attempted to be extracted from the filesystem (which would be slow and/or
    unreliable.)  This is where "mystery" blobs come from; the blob/actor needs
    to be created before everything is known about it.  The child resolves the
    mystery when it reads the structured clone.



## Classes

Interesting operation hierarchies:

* DatabaseOperationBase
  * FactoryOp (isa PBackgroundIDBFactoryRequestParent, OpenDirectoryListener)
    * OpenDatabaseOp
    * DeleteDatabaseOp
  * DeleteDatabaseOp::VersionChangeOp
  * TransactionDatabaseOperationBase
    * OpenDatabaseOp::VersionChangeOp
    * Database::StartTransactionOp
    * NormalTransactionOp (aisa PBackgroundIDBRequestParent)
      * ObjectStoreAddOrPutRequestOp
        * has nested classes: StoredFileInfo, SCInputStream
      * ObjectStoreGetRequestOp
      * ObjectStoreGetKeyRequestOp
      * ObjectStoreDeleteRequestOp
      * ObjectStoreClearRequestOp
      * ObjectStoreCountRequestOp
      * IndexRequestOpBase
        * IndexGetRequestOp
        * IndexGetKeyRequestOp
        * IndexCountRequestOp
    * VersionChangeTransactionOp
      * CreateObjectStoreOp
      * DeleteObjectStoreOp
      * RenameObjectStoreOp
      * CreateIndexOp
      * DeleteIndexOp
      * RenameIndexOp
    * Cursor::CursorOpBase
      * Cursor::OpenOp
      * Cursor::ContinueOp

  * DatabaseOp: exists only for CreateFileOp.
    * CreateFileOp: exists for creating mutable files.  (Journaled appendable
      files that are moz-specific at this point because of filesystem api
      snafu.  Check standards as needed.)

  * TransactionBase
    * NormalTransaction
    * VersionChangeTransaction
  * TransactionBase::CommitOp

### Operation Hierarchy: Who Holds What

FileManager:
* Database: Provided at creation time.  Never cleared.  Exposed via
  GetFileManager().
* DatabaseConnection: Provided at creation time in
  ConnectionPool::GetOrCreateConnection from its Database.  Cleared in Close().
  * Close happens via CloseConnectionRunnable on connection thread.

Does TransactionDatabaseOperationBase's mTransaction hold the database alive?
* TransactionDatabaseOperationBase has TransactionBase mTransaction, exposed via
  Transaction().
* TransactionBase has Database mDatabase. exposed via public GetDatabase().
  Never nulled out.

### State machines

* FactoryOp: Well-documented database/quota-level operations.
  * States: Initial PermissionChallenge, PermissionRetry, FinishOpen,
    QuotaManagerPending, DirectoryOpenPending, DatabaseOpenPending,
    DatabaseWorkOpen, BeginVersionChange, WaitingForOtherDatabasesToClose,
    WaitingForTransactionsToComplete, DatabaseWorkVersionChange,
    SendingResults, Completed.
  * Potentially held by:
    * gFactoryOps
    * FactoryOp::mDelayedOp
    * DatabaseActorInfo::mWaitingFactoryOp

* TransactionDatabaseOperationBase:
  * States: Initial, DatabaseWork, SendingPreprocess, WaitingForContinue,
    SendingResults, Completed.
* DatabaseOp:
  * States: Initial, DatabaseWork, SendingResults, Completed.
* TransactionBase

* WaitForTransactionsHelper
  * States: Initial, WaitingForTransactions, WaitingForFileHandles, Complete.

* Maintenance
  * States (enum class State): Initial, CreateIndexedDatabaseManager,
    IndexedDatabaseManagerOpen, DirectoryOpenPending, DirectoryWorkOpen,
    BeginDatabaseMaintenance, WaitingForDatabaseMaintenancesToComplete,
    Finishing, Complete.

## Life-cycle things

* DatabaseOperationBase
  * NoteActorDestroyed: sets mActorDestroyed, mOperationMayProceed.
    * These cause many of the Run state-dispatched functions to do an early
      return with an error code.

* gFactoryOps
  * List of pending factory ops.  Exists so that ops can know if they should
    wait on another op.  Also so maintenance can know whether it's safe to run
    or not.  (There's also )

## Scheduling

ConnectionPool
  * Start (...TransactionDatabaseOperationBase)
  * ScheduleTransaction(TransactionInfo, aFromQueuedTransactions)
    * If dbInfo->mClosing, queue in mTransactionsScheduledDuringClose and bail.
    * If no thread associated yet:
      * if no mIdleThreads,
        * If we haven't hit our thread limit kMaxConnectionThreadCount yet,
          spawn a thread and assign it to the dbInfo.
        * If there are any mDatabasesPerformingIdleMaintenance (which means
          they have a thread), post dummy runnables to them to make them stop
          doing idle stuff.  No direct assignment happens, presumably because
          it's hard to predict which will come free first, so just wait for them
          to report themselves idle again.
        * If we couldn't create a thread, return false.  If we weren't
          aFromQueuedTransactions, append to mQueuedTransactions.
      * yes, idle threads!
        * Pull the last idle thread (to avoid array shift, presumably), assign.
        * AdjustIdleTimer()
    * (now there must be a thread assigned to the dbInfo)
    * if it's a write transaction
      * if there's already an mRunningWriteTransaction, append to
        mScheduledWriteTransactions and return true.
      * assign to mRunningWriteTransaction, set dbInfo->mNeedsCheckpoint (WAL)
    * Set the transaction mRunning
    * If the transaction has any mQueuedRunnables, dispatch them directly to the
      assigned thread.

  * Dispatch(uint64_t aTransactionId, nsIRunnable* aRunnable)
    * If transaction running, dispatch it to the thread.
    * If not running, append it to TransactionInfo.mQueuedRunnables.
  * Finish(uint64_t aTransactionId, FinishCallback* aCallback)
    * Creates and Dispatch()es a FinishCallbackWrapper runnable to the I/O
      thread.
    * FinishCallbackWrapper invokes the runnable on the I/O thread, then
      dispatches back to the owning/background thread.
    * Back on the owning/background thread, invokes:
      * callback->TransactionFinishedBeforeUnblock()
        * Used by CommitOp to UpdateMetadata.
      * connectionPool->NoteFinishedTransaction(mTransactionId): see below
      * callback->TransactionFinishedAfterUnblock()
        * Used by CommitOp to SendCompleteNotification
  * NoteFinishedTransaction
    * If this was the active write transaction (mRunningWriteTransaction) for
      the db, and if there are any scheduled write transactions
      (mScheduledWriteTransactions), then pull the first one off and call
      ScheduleWriteTransaction().
    * Runs through the blocking transactions, removing the transaction from:
      * if a (the) write transaction, mLastBlockingReads (if same transaction;
        which, based on the single-write-transaction-per-db approach right now,
        means it's always the case)
    * Invokes TransactionInfo::RemoveBlockingTransactions which runs through
      mBlockingOrdered and invokes TransactionInfo::MaybeUnblock which removes
      the transaction from mBlockedOn, and, if now zero, invokes
      TransactionInfo::ScheduleTransaction
    * Removes from mTransactions
    * Invoke NoteIdleDatabase()
  * ScheduleQueuedTransactions(ThreadInfo) AKA "this thread is now idle, try and
    find some use for it from mQueuedTransactions"
    * Put thread into mIdleThreads
    * Iterate over all mQueuedTransactions, invoking ScheduleTransaction,
      breaking


Event results in child:
* DispatchSuccessEvent:
  * (invoked by BackgroundFactoryRequestChild::HandleResponse(overloaded by type),
    BackgroundDatabaseChild::RecvPBackgroundIDBVersionChangeTransactionConstructor,
    BackgroundDatabaseRequestChild::HandleResponse(overloaded by type))
  * Fast-bail to DispatchErrorEvent if the transaction was aborted.
  * Uses CreateGenericEvent
  * Uses IDBRequest::DispatchEvent which is really DOMEventTargetHelper's impl


## Failure Handling

TransactionDatabaseOperationBase defines an Init() method where false can be
