## Classes

* DatabaseOperationBase
  * FactoryOp (isa PBackgroundIDBFactoryRequestParent, OpenDirectoryListener)
    * Subclasses:
      * OpenDatabaseOp
      * DeleteDatabaseOp
    * Potentially held by:
      * gFactoryOps
      * FactoryOp::mDelayedOp
      * DatabaseActorInfo::mWaitingFactoryOp

## Life-cycle things

* DatabaseOperationBase
  * NoteActorDestroyed: sets mActorDestroyed, mOperationMayProceed.
    * These cause many of the Run state-dispatched functions to do an early
      return with an error code.

* gFactoryOps
  * List of pending factory ops.  Exists so that ops can know if they should
    wait on another op.  Also so maintenance can know whether it's safe to run
    or not.  (There's also )
