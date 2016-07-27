# ServiceWorker Stuff #

Investigation questions:
* What is up with things like PropagateUnregister in the context of SWMP/SWMS
  and friends.  Is it just half-implemented?  Not quite sure where one gets
  into the ping-ponging loop that seems to close back on itself.  And certainly
  it seems like the semantics could potentially lead to the unregister job
  running multiple times.

## Participants ##

### By File ###

* RuntimeService: Tracks live workers (dedicated, shared, service) by domain,
  can create shared workers, other book-keeping.
* ScriptLoader: Load top-level and imported scripts in a worker, includes the
  understanding of how to use dom/cache chrome namespace for serviceworker.
* ServiceWorker: Binding exposure of ServiceWorker to main thread; DETH,
  postMessage exposure.
* ServiceWorkerClient: Binding exposure of (currently document only) clients to
  the service worker's global.  postMessage exposure.
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
  jobs.
* ServiceWorkerJobQueue: What the label says.
* ServiceWorkerManager: The big brain, also the nsIServiceWorkerManager
  interface which is where devtools gets registrations from.
  * more
* ServiceworkerManagerChild: Updates local ServiceWorkerManager's understanding
  of the current registrations.
* ServiceWorkerManagerParent: Standard parent/service idiom that calls into
  ServiceWorkerManagerService but with sanity checking of content parent
  authority.  Mainly just for keeping registrations in sync.

## Registrations ##

ServiceWorkerRegistrar keeps registrations in newline-delimited
"serviceworker.txt" in profile directory.  File "header" is version string
(currently 4), followed by repeating blocks of [scope, script, guid].

In the parent process, the SWM directly asks the SWR for the list of
registrations.  (Which the SWR loaded at profile-after-change time.)
