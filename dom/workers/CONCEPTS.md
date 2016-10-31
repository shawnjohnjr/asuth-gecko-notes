## Holders (formerly the poorly named Features) ##

Life-cycle management construct for workers:
* Keeps worker/event loop alive (in WorkerPrivate::DoRunLoop) by having
  HasActiveHolders() return true.
* Worker holders get Notify(status) invoked on them when status changes due
  to calls to WorkerPrivate::NotifyHolders/NotifyInternal.  Status is
  explicitly defined enum in WorkerHolder.h with Pending/Running/Dead being the
  straightforward singleton states and Closing/Terminating/Canceling/Killing
  covering the "gonna shutdown" with the why baked in.
  * True is expected success return value.  Return value of false results in an
    NS_WARNING, otherwise unchecked.

## WorkerRunnable ##

Blinged out runnable helper hierarchy of root class and subclasses.  Provides
PreDispatch/PostDispatch and PreRun/PostRun extension points.  Has target/busy
behavior semantics which impact the busy count.
for subclasses.  Also, subclass variants:
* WorkerDebuggerRunnable: For debugger stuff!
* WorkerSyncRunnable:

## BusyCount ##
Used to stop the worker thread from prematurely canceling.  Tracked on
WorkerPrivateParent<Derived> (in the parent thread), directly manipulated by
ModifyBusyCount.  When it hits zero, the parent's status is checked against
Terminating and if it is, a Cancel() is fired.

Also guards against cycle-collection traversal of mSelfRef.

The WorkerRunnable and its many subclasses care a lot about keeping this
up-to-date.  Example uses:
* AddChildWorker/RemoveChildWorker contribute a maximum busy count of 1 as long
  as there are any child workers.
* Each feature contributes a busy count of 1.
* The presence of an active timer contributes a busy count of 1.

## Garbage Collection ##

## Service Worker Network Retrieval, Storage ##

Service Workers and their importScripts dependencies are stored into DOM Cache.
