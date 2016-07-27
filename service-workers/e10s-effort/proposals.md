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
    spin up a



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
