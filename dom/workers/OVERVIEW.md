# ServiceWorker Stuff #

Investigation questions:
* What is up with things like PropagateUnregister in the context of SWMP/SWMS
  and friends.  Is it just half-implemented?  Not quite sure where one gets
  into the ping-ponging loop that seems to close back on itself.  And certainly
  it seems like the semantics could potentially lead to the unregister job
  running multiple times.

## State Tracking ##

Each ServiceWorkerManager maintains fully instantiated state for all known
registrations/scopes/service workers at all times.  The only dynamic aspect is
that the ServiceWorkerPrivate will not have a WorkerPrivate spun up if nothing
is happening.

ServiceWorkerManager:
* mRegistrationInfos: principal => RegistrationDataPerPrincipal
  * RegistrationDataPerPrincipal (in ServiceWorkerManager.cpp)
    * mOrderedScopes: list of scopes (for glob matching)
    * mInfos: scope => ServiceWorkerRegistrationInfo (registration)
      * ServiceWorkerRegistrationInfo (own h/cpp):
        * mEvaluatingWorker, mActiveWorker, mWaitingWorker, mInstallingWorker:
          (potentially null refptr) ServiceWorkerInfo instances
          * ServiceWorkerInfo:
* mControlledDocuments

SWM::LoadRegistration idempotently brings this state into existence for a given
ServiceWorkerRegistrationData, which is the ipdlh transport type defined in
"ServiceWorkerRegistrarTypes.h".  It accomplishes this via
SWM::CreateNewRegistration and explicitly newing a ServiceWorkerInfo instance
for the (by definition active) registration.

SMW::CreateNewRegistration straightforwardly news a
ServiceWorkerRegistrationInfo and stuffs the provided scope and principal in
(which fully characterize that bit; the SWI gets the worker URL and cache name
that are the other 2 provided fields.)

ServiceWorkerPrivate spins up instances on-demand.  They are owned by
ServiceWorkerInfo instances.


## Registrations ##

ServiceWorkerRegistrar keeps registrations in newline-delimited
"serviceworker.txt" in profile directory.  File "header" is version string
(currently 4), followed by repeating blocks of [scope, script, guid].

In the parent process, the SWM directly asks the SWR for the list of
registrations.  (Which the SWR loaded at profile-after-change time.)


# By File #

* ChromeWorkerScope:
* ...
* RuntimeService: Tracks live workers (dedicated, shared, service) by domain,
  can create shared workers, other book-keeping.
* ScriptLoader: Load top-level and imported scripts in a worker, includes the
  understanding of how to use dom/cache chrome namespace for serviceworker.
* ServiceWorker: Binding exposure of ServiceWorker to main thread; DETH,
  postMessage exposure.
* ServiceWorkerClient: Base binding exposure of (currently document only)
  clients to the service worker's global.  postMessage exposure.
  ServiceWorkerWindowClient is the concrete exposure.
* ServiceWorkerClients: Binding exposure of list of clients known to SW.
* ServiceWorkerCommon: installing/waiting/active bitmask enum
* ServiceWorkerContainer: Binding of navigator.serviceWorker, exposes
  register(), existing registrations, existing controller, etc.
* ServiceWorkerEvents: fetch event, push message event, respondWith handler,
  waitUntil handler, extendable event base logic.
* ServiceWorkerInfo: Canonical representation of the registration for our
  internal gunk, exposed as nsIServiceWorkerInfo.  Populated from the ipdlh
  ServiceWorkerRegistrationData and upconverted to the ServiceWorkerRegistration
  webidl binding.
* ServiceWorkerJob: Job abstraction for the async register/update/unregister
  jobs.  AsyncExecute is the entry point for subclasses' actual logic.
* ServiceWorkerJobQueue: What the label says.
* ServiceWorkerManager: The big brain, also the nsIServiceWorkerManager
  interface which is where devtools gets registrations from.
  * more
* ServiceworkerManagerChild: Updates local ServiceWorkerManager's understanding
  of the current registrations.
* ServiceWorkerManagerParent: Standard parent/service idiom that calls into
  ServiceWorkerManagerService but with sanity checking of content parent
  authority.  Mainly just for keeping registrations in sync.
* ServiceWorkerManagerService
* ServiceWorkerMessageEvent
* ServiceWorkerPrivate: API to trigger events on the SW which may result in the
  WorkerPrivate actually being spun up; also state machine for timeouts and
  termination.
* ServiceWorkerRegisterJob: Register algorithm job, subclasses UpdateJob.
* ServiceWorkerRegistrar: Canonical holder of all registrations: reads/writes
  them from disk.
* ServiceWorkerRegistrarTypes.ipdlh: IPDL types: currently just
  ServiceWorkerRegistrationData, corresponding to ServiceWorkerInfo.
* ServiceWorkerRegistration: Binding exposure for ServiceWorkerRegistration.
* ServiceWorkerRegistrationInfo: Internal stateful registration structure,
  holds/tracks evaluating/active/waiting/installing workers and recency of
  updates, plus state machine logic for those.  Exposed as
  nsIServiceWorkerRegistrationInfo for devtools.
* ServiceWorkerScriptCache: Async Compare() that implements byte-wise comparison
  logic for service worker updates.  Uses: CompareCache, CompareManager,
  CompareNetwork.  Generates new cache names when new script states are
  detected; GenerateCacheName gives them a uuid.
* ServiceWorkerUnregisterJob: Unregister algorithm job.
* ServiceWorkerUpdateJob: Update/Install algorithm job, subclassed by register.
* ServiceWorkerWindowClient: Binding exposure of WindowClient.
* SharedWorker: Binding exposure of SharedWorker (exposes port, postMessage,
  etc.)
* WorkerDebuggerManager: Implements nsIWorkerDebuggerManager for use by
  devtools.
* WorkerHolder: nee WorkerFeature, life-cycle management for keeping alive and
  being notified of termination.  See CONCEPTS.md for more.
* WorkerInlines: Unused? GetJSPrivateSafeish/SetJSPrivateSafeish wrap
  JS_SetPrivate and JS_GetPrivate, but have no callers and PrivatizableBase
  that's used as a tagging type has no descendants.
* WorkerLocation: Binding of "location" in WorkerGlobalScope.
* WorkerNavigator: Binding of "navigator" in WorkerGlobalScope.
* WorkerPrefs: Included by Workers.h as a define-magic payload.
* WorkerPrivate: Core worker logic.  WorkerPrivateParent type exists for parent
  thread things, WorkerPrivate exists for on-worker things.
* WorkerRunnable: Blinged out runnable helper hierarchy to help with common
  worker idioms like managing busy counts.
* Workers.h: Public WorkerPrivate stuff: WorkerLoadInfo, WorkerPreference enum
  (via call out to WorkerPrefs.h), JSSettings.
* WorkerScope: Home to global scope bindings: WorkerGlobalScope,
  DedicatedWorkerGlobalScope, SharedWorkerGlobalScope, ServiceWorkerGlobalScope,
  WorkerDebuggerGlobalScope.  Usually fairly thin on mWorkerPrivate or other
  constructors.
* WorkerThread: nsThread subclass with magic via WorkerThreadFriendKey to limit
  callers to RuntimeService and WorkerPrivate.  Much logic relates to exposing
  the standard XPCOM nsIEventTarget API to work with WorkerPrivate::DoRunLoop,
  invoked by the WorkerThreadPrimaryRunnable (in RuntimeService.cpp).
