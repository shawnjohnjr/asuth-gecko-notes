## Part 8

### Glitch:
* WorkerListener::RegistrationRemovedWorkerThread prototype/decl is added, but
  never actually implemented/defined.
  * WorkerListener is the on-worker-thread implementation.

#### Surrounding Context:

There's 2 interfaces going on:
* ServiceWorkerRegistrationListener: How SWM talks to registrations.  Methods
  are invoked on the main thread.
* ServiceWorkerRegistration::Inner: Implementation indirection so we can switch
  from same-process to remoted but still using the same outer binding class in
  the future.  Currently also handles the main thread/worker thread differences.

There are 2 implementation classes, each of which implement SWR::Inner, they
respectively handle main/worker thread:
* ServiceWorkerRegistrationMainThread: Implements
  ServiceWorkerRegistrationListener itself because it already lives on the main
  thread.
* ServiceWorkerRegistrationWorkerThread: Lives on the worker thread, so it can't
  (sanely) implement ServiceWorkerRegistrationListener itself.  Instead, we have
  WorkerListener:
  * WorkerListener: Created by ServiceWorkerRegistrationWorkerThread to handle
    the main-thread/worker-thread dance.

#### remove() changes
These are the changes that happened in the remove() space:

Main Thread:
* ServiceWorkerRegistrationMainThread::RegistrationRemovedInternal introduced,
  new home to the conditionalized InvalidateServiceWorkerRegistration() call.
  * Newly invokes StopListeningForEvents().  Previously that was only called by
    ClearServiceWorkerRegistration which was in turn called by SWR's destructor
    and its DOMEventTargetHelper::DisconnectFromOwner() override.  This makes
    sense because removal is the end of the line.
  * Newly nulls out mOuter, with nice comment explaining about the cycle
    breaking.
  * This was done so that...
* ServiceWorkerRegistrationMainThread::RegistrationRemoved uses
  NewRunnableMethod to defer execution of the internal method so that other
  runnables that update the registration state don't get caught out.
  * Now that most of the SWR::Inner implementations throw if !mOuter, this
    deferral makes sense because previously moot API calls would work.  Now they
    throw, likely resulting in annoying test breakage.

WorkerThread impl on Main Thread:
* WorkerListener::RegistrationRemoved, which was an inline assertion that it was
  invoked on the main thread, is now an override with separate implementation.
  * It invokes the new runnable to invoke RegistrationRemoved on the worker
    thread:
* RegistrationRemovedWorkerRunnable introduced to
  invoke ServiceWorkerRegistrationWorkerThread::RegistrationRemoved on the
  Worker thread.


WorkerThread impl on Worker Thread:
* ServiceWorkerRegistrationWorkerThread::RegistrationRemoved prototype/decl
  introduced in header.
* ServiceWorkerRegistrationWorkerThread::RegistrationRemoved just nulls out
  mOuter.
* The confusing WorkerListener::RegistrationRemovedWorkerThread prototype was
  added.
* ServiceWorkerRegistrationWorkerThread::Notify nulls out mOuter, with a comment
  that explains why posting a runnable doesn't work.

#### Conclusion

Options:
* SWRWT::RegistrationRemoved ended up being what
  WorkerListener::RegistrationRemovedWorkerThread thought it was going to be.