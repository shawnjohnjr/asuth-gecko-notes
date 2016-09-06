# Discussion at MoTo #

## Notes ##

Creating worker: main thread.
SWM: A thread in parent.  Probably not PBackground worker thread.
* PBackgroundWorker: "I'm alive!".  fire Event!
(This being a rendezvous dance.  Async creation of worker in child, need to wait
for worker to show up in the parent before dispatching events to it.  Probably
want backstops of child saying: "didn't work", and noticing if the ContentParent
died.  Book-keeping-wise probably want to track from the moment of the request
anyways, which simplifies things.)

PSharedWorkerManager.
PWorker:
* Need to create types for the remoting of the existing service worker events.
PServiceWorkerEvent (I just made up name, seems to work)
* As I proposed at least, we use the constructor-is-the-request idiom, and the
  self-delete-request-carries-the-result idiom.  Event is assumed to have died.
* Needs to be a WorkerHolder.


RemoteWorkerService:
* In non-e10s case needs to spawn within the same process.  Should bypass
  directly to RuntimeService to the extent possible.
* In remote case use PContent (for now).  It's regrettable but there's no
  PChildBackground or anything else to give us a target thread to talk to.

Want functor-esque decision-making process for where to place the ServiceWorker.
Not just hardcode "this process for ServiceWorkers".  Want to be able to spin
ServiceWorker up in the same process its requesting content is coming from.
(Implies intercept request should have associated ContentParent.)

We want ServiceWorkerManager in PBackground or even its own thread.  Unclear
what the current level of coupling is other than nsIDocument.

Track idle in parent and issue terminations from there.  Need to wait on clean
shutdown of the worker, though, for things like DOM Cache that might have in
flight stuff they're finising.

baku tried to refactor postMessage everywhere, but things like mediastream
couldn't be serialized, so he had to stop/give up.

Leave RuntimeService as a low level toolkit.

## Gameplan ##

* Create RemoteWorkerService
  * It is responsible for book-keeping of locations of ServiceWorkers and
    SharedWorkers and their naming scopes.
* Create PSharedWorkerManager
  * SharedworkerManagerParent will talk to RemoteWorkerService to cause it to
    spin up a SharedWorker somewhere if it doesn't exist.
* Update RuntimeService
  * RS::CreateSharedWorker creats

# NEW PROVISIONAL GAMEPLAN #

## Change: bkelly's client manager ##
(in dom/clients)

bkelly proposes:
* Actor for each client.
* Mechanism for delivering postMessage to clients (from client.postMessage call)
* InnerClient::ReportToConsole: Simplifies error reporting.


## Participants ##

Parent Process:
* ServiceWorkerManagerService (PBackground):

Content Processes:
* ServiceWorkerManager (Main): Local interaction, uses PBackground even in
  single process.

## How Do Bindings Expose Things ##

* ServiceWorker
  * creation:
* ServiceWorkerRegistration
* ServiceWorkerContainer (as navigator.serviceWorker)
  * ServiceWorker? controller:
  * Promise<SWR> ready:
  * register() => SWR:
  * getRegistration() => SWR/undefined:
  * getRegistrations() => SWR[]:
* ServiceWorkerMessageEvent (client-received postMessage)
* ServiceWorkerGlobalScope:
  * (clients)
  * (registration)
* Clients: bkelly handling https://bugzilla.mozilla.org/show_bug.cgi?id=1293277
* Client: bkelly handling https://bugzilla.mozilla.org/show_bug.cgi?id=1293277



## Protocols ##

PServiceWorkerManager:
* parent:
  clientPostMessage:
* child (main):
  PServiceWorker: spawn a ServiceWorker
* child (background):

PServiceWorkerRegistration:

## Id's ##

* Window ID's: Allocated by ContentChild.cpp:NextWindowID() as a 64-bit globally
  unique value that is never recycled.
* Client id's:
  * document: nsIDocument self-issued GetOrCreateId via GenerateDocumentId which
    is a UUID with the {}'s and the null stripped off.  (Created and saved off
    to mInterceptedDocumentId in nsDocShell::ChannelIntercepted.)
  * workers: We don't expose these yet.
* TabId's: Allocated by ContentProcessManager::AllocateTabId

* ScopeKey: Vestigial; PrincipalOriginAttributes has mooted this since
  everything that matters is baked into the suffix.  As such, this is now just
  the origin (which includes the suffix).
* ServiceWorker cacheName: A UUID generated to name the Cache when we load the
  serviceworker script and stash it in the cache.

# Implications of Multiple ServiceWorkers #

## Fallout notes

* SharedWorkers still want to be actively managed by the parent process.
* ServiceWorkerManagerParent still wants to coordinate the job queue and
  maintain the related invariants.

## Inherent Implementation Fallout

* ServiceWorkerInfo owns the mServiceWorkerPrivate.  

## Potential Performance Complications

* If we have a SW for origin foo in process A and open/intercept foo in process
  B, do we want to reuse the SW in A?
  * Being able to do this means the implementation needs to be able to do
    cross-process things; can't simplify everything is same-process.  But that's
    probably an untenable simplification anyways.  We don't want to be spinning
    up N SW's for super-prolific iframes like Google +1/Facebook like buttons,
    etc.

## Conclusions for Gameplan (Mine) ##

* Refactor ServiceWorkerManager job queue and state machine enforcement into the
  parent.
* Navigation intercepts happen in the parent.  This causes the registration data
  to be made available.

* Binding updates/events are addressed by coherently sending updates to the
  child ServiceWorkerRegistrationInfo which then applies the modifications to
  its ServiceWorkerInfos and possibly generates events, etc.
  * PServiceWorkerRegistration corresponds to this, allowing the life-cycle of
    the actor serve as the book-keeping for interest in the parent.

* Where/When to spawn controlled by policy provided by the parent to the child
  with the registration.  Can say "route to parent", "spin up to N of your own".

* Process holding open still needs to exist for cases like push and background
  sync where it's possible for it to make sense for the SW to exist in a process
  without anyone else holding alive.
  * Create something like concept of HeadlessClient(Id) which can serve to
    justify opening the content process.  And potentially to cascade the
    justification, like background sync => service worker => shared worker;
    probably important to the shared worker that its authority to be alive comes
    from the headless client.

### Notes:

* maybe have actor for ServiceWorker too.
  * bkelly said we maybe have to because of the need to be able to still talk
    to it even after it's gone redundant?  Cases:
    * "Install" installFailed (step 14): the installing work gets marked as
      redundant and install aborts, but we don't seem to explicitly call for
      terminating the worker.
    * "Install" step 16, if there was a waiting worker, we set it to the
      redundant variable and terminate it.  (Then in step 20 we update its
      state for good measure?)
    * "Activate" step 3, terminates existing active worker (and uses variable),
      updates state afterwards in step 7.

* no strong opinion id-wise.



### Tricky Stuff ###

* UpdateJob's use of CheckScriptEvaluation needs to cause the worker to be
  spawned.  So this needs its own special event type.
* We need a remoteable

# Blank Slate Brainstorming #

## Blank Slate Approach for Worker Management ##

WorkerCoordinator:
* Knows current SharedWorker and ServiceWorker instances and where they live,
  characterized by identifying key.
* When asked to instantiate such instances, causes them to be spun up.
* When told they don't matter anymore forgets about them.
* Can route events to ServiceWorkers.  (Doesn't really care about ServiceWorkers
  that much, but for protocol coupling reasons, i )

## Intrusions of Reality ##

* Channels get created on the main thread, so there needs to be a main-thread
  source of truth.

## Map to Reality ##
