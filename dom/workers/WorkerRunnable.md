See CONCEPTS.md first for details on us and BusyCount.

## Hierarchy ##

Overview of WorkerRunnable:
* Blinged out nsIRunnable/nsICancelableRunnable variant for when the source or
  target of a runnable is a worker.
* For on-worker runnables, provides access to the worker's JSContext.
* Has target/busy behavior semantics which impact the busy count.  This is the
  main reason you really want to use this class, since normal runnables wrapped
  via ExternalRunnableWrapper will not affect the busy count.
* Provides PreDispatch/PostDispatch and PreRun/PostRun extension points, all of
  which mainly exist for the benefit of subclasses to add assertions and perform
  common setup.
  * Note that the base class provides behavior for all, so subclasses may end up
    subclassing PostDispatch with a NOP, for example, in order to disable the
    pre-existing PostDispatch thread assertions.
* If used directly:
  * Asserts that it's on the parent thread if behavior is
    WorkerThreadModifyBusyCount, asserts it's on the worker thread otherwise.

WorkerRunnable Subclasses:
  * WorkerSyncRunnable: A runnable to be dispatched to a spinning sync loop on
    the worker thread.  The caller needs to have the nsIEventTarget of the sync
    loop for this to be different from a normal WorkerRunnable.
    * MainThreadWorkerSyncRunnable: A WorkerSyncRunnable subclass for when the
      sender thread is the main thread and wants constructor and pre-dispatch
      assertions to that effect for free.
    *
  * WorkerControlRunnable: Basically a tagging interface for runnables that want
    to cut the line.  Put into the WorkerPrivate's mControlQueue which is
    drained at a higher priority than the normal NS_ProcessNextEvent queue.
    * MainThreadWorkerControlRunnable: Control runnable for when the sender is
      on the main thread and wants pre-/post-dispatch assertions to that effect.
  * MainThreadWorkerRunnable: For when the sender is on the main thread and
    wants constructor and pre-/post-dispatch assertions to that effect for free.
  * WorkerSameThreadRunnable: Dispatched from the worker to itself.  Alters
    base class behavior so that the busy count is still modified
    (WorkerThreadModifyBusyCount) but asserts worker thread instead of parent
    thread.

Other Runnable variants:
  * WorkerMainThreadRunnable: Sync-call-to-main-thread runnable base class.
  * WorkerProxyToMainThreadRunnable: Bounce-to-main-thread-and-back runnable
    variant with aptly anmed RunOnMainThread() and RunBackOnWorkerThread()
    methods to implement.  Uses a "SimpleWorkerHolder" to keep the worker alive
    until
