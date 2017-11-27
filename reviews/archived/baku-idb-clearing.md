### patch delta bookkeeping ###

#### removed-ish ####

* ResetOrClearOp
  * DeleteFiles. : Removes storage/ and storage.sqlite
  * DoDirectoryWork: RemoveQuota(), ResetOrClearCompleted()
  * GetResponse
* ClearReqeustBase
  * DeleteFiles(QM, PersistenceType): Enumerates origin directories, skips
    non-matching ones.

#### added-ish ####

* ClearContentOp created.
  * struct OriginData  { mFile, mOrigin, mGroup } to hold CreateOriginList
    results
  * nsTArray<OriginData> mOriginList: CreateOriginList results
  * nsTArray<RefPtr<DirectoryLock>> mLockList
  * mMonitor: coordinate the I/O thread and owning thread activities
  * Listener: does the book-keeping to wait for the locks
    * mCount: tracks number of locks it's waiting on
    * mCompleted: fire-once boolean so one of mOp->DirectoryLocksAcquired() and
      mOp->DirectoryLocksFailed() is invoked at most once.
    * **q: locks are saved off, yes**
  * CreateOriginList: created out of ClearRequestBase::DeleteFiles, it uses
    hard-coded "^http" and "^file" prefix origin matchers.  These get https as
    well as http.
