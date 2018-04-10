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

Blinged out runnable helper hierarchy of root class and subclasses.  Key bits:
* Provides access to the worker's JSContext* and WorkerPrivate* in WorkerRun().
* Provides explicit management of the busy count for the worker.  This matters

* Provides
PreDispatch/PostDispatch and PreRun/PostRun extension points, all of which
mainly exist for the benefit of subclasses to add assertions and perform common
setup.  Has target/busy behavior semantics which impact the busy count.  See
WorkerRunnable.md for more info.

Normal runnables will be wrapped in ExternalRunnableWrapper instances when
targeted at a worker.  Cancel() will be propagated if the runnable implements
nsICancelableRunnable.  These use the WorkerThreadUnchangedBusyCount behavior
which means they do not modify the worker's busy count,

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
* Each WorkerHolder contributes a busy count of 1.
* The presence of an active timer contributes a busy count of 1.

### Versus Holders ###

Holders prevent the event loop from terminating.  The busy count prevents the
worker from getting garbage collected.  (AKA getting canceled from the busy
count going to 0.)

Note that a Holder can also boost the busy count.
(https://bugzilla.mozilla.org/show_bug.cgi?id=1362444 introduced the ability to
have the holder NOT boost the busy count for cases where we want to respond to
worker shutdown but not prevent it.  This partially explains why things are a
bit confused; a lot of things were previously conflated.)

## Worker Shutdown / Termination ##

### Pending WorkerRunnables

The spec dictates that invoking Close() should clear the event queues and
prevent further tasks from being enqueued.  We do this.  We invoke Cancel() on
the runnables.

The mechanism is that existing runnables are canceled via:
- WorkerPrivate::mCancelAllPendingRunnables gets set to true, which causes
  WorkerPrivate::AllPendingRunnablesShouldBeCanceled() to return true.
- WorkerRunnable::Run() checks AllPendingRunnablesShouldBeCanceled() in its
  (wrapping) Run method, and invokes Cancel() instead of Run().  (There is some
  sketchy re-entrancy protection, however.)
- ClearMainEventQueue drains the current event queue by setting
  mCancelAllPendingRunnables and invoking NS_ProcessPendingEvents() which runs
  the (opaque) platform event queue machinery so we can clock all the runnables
  out, canceling them.

And future runnables will fail to dispatch because:
- WorkerPrivate::DispatchPrivate checks ParentStatus() and errors out if our
  status isn't Pending or Running.

## Garbage Collection ##

Sorta sketchy.
