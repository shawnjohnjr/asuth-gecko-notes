Logic relates to exposing the standard XPCOM nsIEventTarget API to work with
WorkerPrivate::DoRunLoop, invoked by the WorkerThreadPrimaryRunnable (in
RuntimeService.cpp).

### WorkerThread::Dispatch ###

Weirdness:
* If invoked off the worker thread and !mWorkerThread, nsThread::Dispatch is
  still invoked, presumably resulting in the runnable running on the bare
  nsThread after WorkerThreadPrimaryRunnable has completed its run.

### Assertion Logic ###

mOtherThreadsDispatchingViaEventTarget is a counter only accessed while holding
mLock.  It's incremented when the custom Dispatch method is handing things off
to the superclass nsThread::Dispatch.  
