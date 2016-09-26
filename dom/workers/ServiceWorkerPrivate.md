## ServerWorkerInfo ##



## ServiceWorkerPrivate ##

### StoreISupports / RemoveISupports ###

(sufficiently documented by mSupportsArray comment, restating for learning)
Main-thread-only API to add/remove cycle-collected references to nsISupports
things on the main thread.

Only allowed Cleaned out by the main-thread-only TerminateWorker()

### Event Dispatch ###

Events where the caller gets/wants a result.
* FetchEvent:
  * Explicit respondWith, required in dispatch turn of the event loop, although
    since you can return a promise, it doesn't really matter.
* LifeCycle
* CheckScriptEvaluation (uses lifecycle callback mechanism)

Events where caller does not care about the result (run for side-effects):
* Message
* Push, Push Subscription Change
* Notification


#### Lifecyle ####

* LifeCycleEventCallback: A runnable that has a SetResult(bool) method intended
  to be invoked on the worker, with the runnable then dispatched to the main
  thread.
* LifeCycleEventWatcher: On-worker interposed PromiseNativeHandler.  In the
  case the event completes, it sets/dispatches the callback.  Also registers as
  a WorkerHolder, and if it hears the Worker got terminated, it sets false and
  dispatches the callback.
* LifecycleEventWorkerRunnable: Created on main thread, dispatched to worker
  where it creates a LifeCycleEventWatcher to watch over the dispatch.

### aLoadFailedRunnable arg of SpawnWorkerIfNeeded ###

A runnable to be dispatched to the main thread in event of
NS_FAILED(loadInfo.mLoadResult) for any of the top-level script loads.  These
are transport failures, not execution failures.  Logically, we expect these to
matter only when a) initially fetching the scripts from the network and b) for
paranoia.

#### Control Flow Investigations ####

The existing use-cases are then:
* Not used, assumed inductive cover from cache:
  * SendMessage
  * CheckScriptEvaluation
  * Notification
  * Push
* Paranoia:
  * SendFetchEvent: Uses to ensure a reset occurs.
* Used to approximate the effect of if the LifeCycleEventCallback had
  SetResult(false) called on it by using a runnable-method to directly invoke
  what the custom runnable accomplishes.
  * SendLifeCycleEvent:
    * ServiceWorkerRegistrationInfo::Activate: Used to trigger
      FinishActivate(false) instead of bouncing through the
      ContinueActivateRunnable to propagate the SetResult.
    * ServiceWorkerUpdateJob::Install: Same deal, but with the method being
      ContinueAfterInstallEvent.
