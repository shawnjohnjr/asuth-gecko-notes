# Overview #

## Goal ##

Expose ServiceWorkerPrivate as PServiceWorkerInstance, spun-up on demand from
the parent.  Ideally this instance will be as self-contained as possible, having
all defining information shipped over in the constructor rather than depending
on the registration existing locally.

We are doing this because the existing ServiceWorkerPrivate implementation
largely constitutes a clean abstraction boundary from the rest of the system.
This enables converting the ServiceWorkerManager to reside in the parent process
only.  Interception will occur there, and Service Worker instances will be spun
up in content processes as needed, living in isolated worlds that interact with
remote-able services.

## Simplifications ##

Logic in the parent and child will both be on their respective main-threads via
PContent.  This is because other than the awkward PBackground-managed
PServiceWorkerManager (which is only about propagating registration changes
across all children and has to bounce everything to the main thread), all
existing logic is main thread.

## Dependent Changes ##

Correctness depends on the following other issues being resolved.

### Parent Intercept ###

These changes assume all events for ServiceWorkers will be dispatched from the
parent process, including fetch events.  This necessitates that event
interception occurs in the parent.

### Client APIs ###

https://bugzilla.mozilla.org/show_bug.cgi?id=1293277 is expected to provide
all functionality related to the ServiceWorkerClients and ServiceWorkerClient
bindings.  Additionally, this bug should provide:
* A multiprocess-safe Client.postMessage implementation that is responsible for
  delivering the request to the ServiceWorkerManager in the parent process.
* The reciprocal implementation of ServiceWorker.postMessage.  It's a slight
  scope creep but seems appropriate because the MessagePort infrastructure
  cannot currently convey the required `source` parameter and it arguably makes
  sense to normalize ServiceWorker to exist in a manner analagous to Client.

For the stop-gap (see test verification), ServiceWorker.postMessage should work
because the page will be in the main process and the binding will automatically
be doing the right thing.  However, Client.postMessage will not operate.

### Remoted Registration State ###

The ServiceWorkerRegistration binding needs to have its state propagated across
all processes.  Currently, state is held locally in each process with only high
level registration added/updated/removed propagated.  Event triggering is also
required.  This potentially overlaps with the Client API changes if that
mechanism becomes involved with the ServicerWorker binding representation.

For the stop-gap (see test verification), the page will be in the main process
and so will see a correct representation.  The binding in worker instances,
however, will only observe correct values for the "active" worker as informed by
PServiceWorkerManager propagation.

## Test Verification ##

### Main Process Stop-Gap ###

Most of the dependent changes have to do with remoting state.  We are able to
sidestep most of this complexity by running the browser with
"browser.tabs.remote.autostart" set to false so that all tabs run in the main
process.  However, we still forcibly remote the ServiceWorkerPrivate into a
child content process for isolation purposes.

# Design #

## Participants ##

* ServiceWorkerPrivate: Continues to exist as API exposure to
  ServiceWorkerManager and friends and maintainer of the notion of when the
  worker should be spun up and terminated due to idle-ness.  However, instead of
  constructing a WorkerPrivate and dispatching runnables to it, a
  ServiceWorkerInstanceParent (PServiceWorkerInstance) is spawned and messages
  are sent to it instead via helpers on ServiceWorkerInstanceParent.
* ServiceWorkerInstanceSpawner: Simple parent-only singleton that interacts with
  ContentParent to spawn remote ServiceWorkerInstances.  This includes spawning
  the (for now) dedicated service worker process while providing the potential
  for alternate spawning strategies in the future.
* ServiceWorkerInstanceChild: Spawns the WorkerPrivate on creation and tears it
  down when the child actor is deleted.
* ServiceWorkerInstanceParent: Adapt ServiceWorkerPrivate calls to the protocol
  and process and report worker-level errors and other information.
* ServiceWorkerEventChild: The new home of the per-message worker logic that
  originally resided in ServiceWorkerPrivate.cpp.
* ServiceWorkerEventParent: Tracks per-event state (callbacks, etc.) and
  handles both completion notifications or event-specific responses such as
  fetch's respondWith.



## Arbitrary Decisions By Me ##

Name it PServiceWorkerInstance rather than PServiceWorkerPrivate.  Rationale:
* "ServiceWorker" is already the name of our WebIDL binding.
* "Private" will now be leaking beyond the directory into PContent and friends.
  Although there is some naming consistency with the base WorkerPrivate
  construct, arguably it'd be more confusing to others and the leakage moots any
  benefit.

Spawn/reap management.  Pre-patch, ServiceWorkerInfo and ServiceWorkerPrivate
instances exist at all times.  It's only the underying WorkerPrivate that is
instantiated on demand,

## Event Types ##

Synthetic events we need:
* evalutateScript (from SWP::CheckScriptEvaluation)
Explicit ExtendableEvents:
* install (lifecycle)
* activate (lifecycle)
* fetch
* (foreignfetch is not yet supported)
* postMessage
(Extended via spec)
* push
* notification

## Control Flow Changes ##

### Scheduling Soft Updates ###

TODO.

Functional events call for a soft-update timer-check to be performed following
the event proper.

Pre-patch, this was accomplished by use of the
ExtendableFunctionalEventWorkerRunnable runnable which decorated
ExtendableEventWorkerRunnable's PostRun to dispatch a RegistrationUpdateRunnable
with aNeedTimeCheck=true to the main thread.
ServiceWorkerRegistrationInfo::MaybeScheduleTimeCheckAndUpdate would then invoke
book-keep the pending check on mUpdateState and invoke SWM::ScheduleUpdateTimer.
mUpdateState is a memory-only flag that manifests in the check-and-clear
SWRI::CheckAndClearIfUpdateNeeded used by SWM::UpdateTimerFired.  With the
latter being a consequence of the ScheduleUpdateTimer's 1sec nsITimer it created
firing.  The duplicate check suppression currently happens in
ScheduleUpdateTimer when arguably it could make sense to suppress earlier in
SWRI.

Post-patch, ExtendableFunctionalEventWorkerRunnable and
RegistrationUpdateRunnable are simplified to register themselves as native
promise handlers that fire upon any resolution of the underlying event.
XXX check versus callback API

### Push Error Reporter ###

TODO.

Pre-patch existed in SWP.cpp as the PromiseNativeHandler for the event with some
synchronous failure handling built-in.  It reported an nsAString messageId and
uint16_t reason (which ErrorResult is capable of conveying) to the pushservice
via nsIPushErrorReporter.

Post-patch, the error handling is pushed upward into the push service.

# Interaction Points #

## Ownership ##



## Code References ##


Callers of ServiceWorkerInfo::WorkerPrivate via dxr
https://dxr.mozilla.org/mozilla-central/search?q=%2Bcallers%3A%22mozilla%3A%3Adom%3A%3Aworkers%3A%3AServiceWorkerInfo%3A%3AWorkerPrivate%28%29+const%22

* ServiceWorker: PostMessage uses it.

* ServiceWorkerClients.cpp:WebProgressListener: Used by its OpenWindowRunnable
that services ServiceWorkerClients::OpenWindow.  Necessary for the
SWP::{Store,Remove}ISupports mechanism which wants a (cycle collected)
main-thread owner for the worker-triggered request.

* ServiceWorkerManager:
  * References are all through the ServiceWorkerInfo's WorkerPrivate() getter,
    usually through GetActiveWorkerInfoForScope, and all just for event
    delivery:
    * SendPushEvent, SendPushSubscriptionChangeEvent relays
    * SendNotification is a multiplexing helper used by exposed suffix-variants
    * DispatchFetchEvent hands it off to the ContinueDispatchFetchEventRunnable.

* ServiceWorkerUpdateJob:
  * CheckScriptEvaluation call. *needs specialized event as noted elsewhere*
  * Call to SendLifeCycleEvent("install") in the installing lifecycle, is fine.

* ServiceWorkerRegistrationInfo:
  * Clear() calls NoteDeadServiceWorkerInfo() which is an assert with an
    explicit TerminateWorker() call.  (Can be remoted.)
  * Activate() sends life-cycle event, is fine.
  * IsIdle() calls WorkerPrivate's IsIdle.  The remoter can capture this in its
    understanding of the outstanding events.  This is consistent with the
    current implementation which just tracks token count and treats the idle
    token as idle.  (As in, lingering things unsecured by a waituntil get the
    shaft.)

## ServiceWorkerUpdateJob ##
