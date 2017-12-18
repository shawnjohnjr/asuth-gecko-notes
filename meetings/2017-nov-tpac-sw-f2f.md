# TPAC Tuesday, Nov 7th
F2F Issue
https://github.com/w3c/ServiceWorker/issues/1206

## Implementer Updates
### Microsoft
* fetch and SW enabled in internal builds
* should be available in november/december insider builds by default.
* flags allow enabling in october creator's update
  * there are a ton of bugs that have been fixed since those builds.
* still going to turn on their multi-instance implementation as discussed
* they have some push issues (but we're punting discussing push at this meeting
  because the relevant people aren't here.)

### Chrome
* Push to normalize tests with WPT tests.  Largely integrated/migrated all tests
  to WPT and now write WPT tests by default.
* Work to speculatively start up serviceworkers from the omnibox.
* Other performance work.  Process communication.  How to split up work between
  processes.

Performance (Alex Russel, Jake Archibald):
* Tough call in the long tail for sites on slow mobile.

### Mozilla
* Re-architecture.
* updateViaCache (57 or 58, unsure which)
* Worker Start
* Performance Timeline
* Really close on Readable Streams, but doesn't look like it's going to make it
  for 58.

### Facebook
Performance:
* Out of the box, it's a regression.
* Unlocks the capability to make things much quicker.
* Loads that use the SW path are way quicker.  On the order of 20% faster.
* Have to hit the ideal path enough.
* aside: A lot FB can ship once Firefox ships Readable Streams.

## Traversing Full Issue List
### Multiple Instances
Webkit:
- option on the table: Push, Notification, Fetch in different instances.
- haven't committed to a model yet re: multiple instances.
- rationale ties back to battery life, cpu, other restrictions.
- Apple internal nitty gritty issues.
- iOS has evolved to make expensive things be rare.
  - Trying different models to make expensive things stay rare.

Facebook:
- Currently relies on knowing that the page actually opened.  Tying the fetch
  back to the notification click.  Natural to know that the click resulted in a
  fetch.

Jake:
- continue status quo of see what happens as we move further along.
  - Webkit, MS agree.

Webkit:
- Could end up in a world where notificationclick and there's never a push prior
  to that.
- Jake: Because discussion of declarative push that allows a notification to
  happen directly without involving serviceworkers.
  - MS also interested in.  Facebook cool with it.
  - Jake: Chrome historically resistant for cases like "Let's meet at 8 o'clock
    at ..." that truncates and that when you click on it the app can't load.
    - But potentially not an issue because the payload could include more data.
    - Alex: sorta off-topic, punt.

### Clients API
https://github.com/w3c/ServiceWorker/issues/1091

Jake:
- (Provide information about which client made the fetch.  If it's a navigation,
  which client is going to be replaced, and which clients will be created by
  this fetch.)
- Issue 1206: clientId (source), resultingClientId (client the page goes into,
  possibly about:blank that already exists), replaesClientId (client that would
  go away or go into the history if navigation is successful.)
  - (Where about:blank has the nuts global reuse scenario.)

Jungkees:
- 2 cases:
  - static iframe, initial about:blank in the iframe case has the same origin
    and navigated to the same origin global.
  - window.open with the same circumstances
- are these the APIs we really want to expose to developers for their use-cases?

Jake:
- Choices:
  - Pretend weird stuff doesn't happen.
  - Show exactly what happens and they need to know.
- History seems to be to let developers mess with the global even before the
  page loaded.

Alex:
- parent creates iframe, iframe implicitly has same-origin, document.write
  implicitly has origin
Jake:
- no, there is a src, and it's the same origin.  It's going to open a new
  document either way.  There's going to be a clobber either way between
  document.write and the navigation.
- it was an (ergonomic) feature to pretend that iframe loading is asynchronous.
  Worried that if we land features on top of that that pretend that isn't
  happening, we're going to create weird edge-cases.
- can't go back in time, but if we could, there'd be a promise or what not.
Jake/Alex discussion.  Jake:
- Want to surface in a predictable way.  Even if there's an error, it happens
  during development, not an edge-case on a live site that bites you.
- Spec-wise we're in a tricky place.  More important to indicate what the
  browser does than what the spec says.  Could end up with the client API
  lying to conform with the spec, but that's not what's actually happening.
  Don't want the two to get out of sync.

Jake:
- Reserved clients summary: may not come into being if there's a
  content-disposition that triggers a download, or there's any type of
  navigation failure.
- We discussed exposing an actual reserved client instance that would buffer
  postMessage/etc. which introduced edge-cases.
- Concerns by Ben that we'd need to fudge the spec to make this work.  Ask
  Ben for confirmation of viewpoint.

bkelly:
- Fairly unusable for anything but queueing the postMessage.  Most other things
  provide default values or just reject if you try and use them.
- Likes proposal of clients.get() delaying until the client comes into
  existence.  Doesn't prevent changing it later to have speculative.
falken (referred to by junkess):
- Agrees, easier not to expose.  Likes proposal.

Jake:
- (summarizing wins of 1216)
Alex:
- What do we lose?
Jake:
- Deadlock case if you write a fetch handler that first asks for the resulting
  client before returning the fetch.
  - Won't deadlock in the about:blank case where the client already exists.
  - This is good because the deadlock case is the common case.
bkelly:
- Likes the concept of execution ready Jungkees raised for the spec.  Will keep
  reserved client id.
- Doesn't get us out of the edge-case, because the about:blank clients are
  already execution ready.  So querying them returns about:blank and then will
  change later.
Jake:
- (summary): original rationale for the reserved client was to provide context
  about the frame/window that will be opened.  Happy that that's somewhat
  satisfied by about:blank already being exposed.
bkelly:
- Firefox optimization to defer creating the global if we don't need it.  We
  try and hide it from script.  (Like how we hide bfcache to make it not
  observable.)
- bkelly implementing similar optimization so that if you touch the client the
  global will be created.  So like if postMessage() to it.
Jake:
- Would a simple get() create the global?
bkelly:
- (impl overview): Client instances independent from the global.  Synchronously
  creates the inner window when it needs it.
- We don't create the environment settings object when the spec says to, we
  lazily wait.
- But bkelly is creating the client when the spec says to do it.
- Mentioning the optimization since it might affect them as well.
falken:
- Might be easier to defer to when they attach ServiceWorkerContainer to the
  window.
bkelly;
- execution ready is when the fetch has completed and you have the URL.  For
  Example when you set the base URL.
falken:
- That might be "navigation commit".
jungkees:
- Set execution ready flag right before we can execute the script.  Right after
  document and window global are created, ?right before invoking the script?.
  So right before the environments are ready.

Jake, writing in issue:
- `clientId`: made the request
- `resultingClientId`: becomes the client
- `replacesClientId`: replaces some client
Alex:
- can I have all three?
Jake:
- Yes: Click a link with a target that targets an iframe.  Existing client
  replaced, new client created, outer document still exists.

bkelly:
- worklets.
- page does the module load, including subresource fetches, and then (magic).
- So there would be no client id's involved because they're subresources.
  Because the owner does the fetches and places them in opaque origins.
- It's a new case where a subresource is creating globals.
- There's an added weirdness about what the import() method should do.  It's
  not related to the client.
jake:
- I think spec says it gets fetched once and then instantiated multiple times.

falken:
- Did we decide that we need all three?
  - Wants to implement resultingClientId first.  Don't implement the others yet.
  - Expose clientId right now, but it's just null.
jake:
- Probably okay if developers can tell it's not implemented and we can implement
  it in the future without breaking code.
- Such as returning null.
- (what about FB?)

n8s:
- (eh)

jake:
- Implementation priority is `resultingClientId`.
- Will check that returning null for the others won't doom us.

### Navigating Reserved/In-Flight Navigations: Issue 1123
falken: Agreed everything moots.
bkelly/others: use-case of falling back to a fallback site if things are slow or
  whatever.

### Web Platform Tests
bkelly:
- Thinks there may be holes in the WPT for Cache API, mentioning it for the new
  implementers.

### Resurrection
Jungkees:
- Reason for originally introducing it was that deletion is not synchronous.
  There's the issue of register/unregister/register/unregister/register need
  to get queued up.  But resurrection allowed it to (effectively) coalesce.
  (approx) Plus there's the idempotency of redundant register() calls in the
  first place.
bkelly:
- Issue of same version of script stomping on the storage.
- We could set a "reinstallation" flag.

What about unregister.

n8s:
- Will unregister if the auth cookies disappear and so there's inconsistency
  with SW's.  Can happen because of webext's.
Alex:
- Do you clear all local storage
n8s:
- Yeah.
Alex:
- What about "clear site data"?  It actually disconnects all storage from the
  tab.
n8s:
- No?  Think it's orthogonal.  (Alex response second sentence above.)

Jake:
- Is the problem that unregister than register is a no-op?

Ali:
- unregister/register as an explicit force update.  unregister/register is
  (unintuitive)

bkelly:
- Introduce a "reactivate" event.
Jake:
- Why?
bkelly / n8s:
- Re-claim.  Re-install?
Alex:
- extendable?
bkelly / n8s:
- Yes.  bkelly: Hard to implement.
...
bkelly:
- Last one will end up at waiting

falken:
- Could block register on the last one going away.
others:
- Could deadlock.

various:
- Digression about resurrecting the registration versus the (versioned) worker.

### ServiceWorker registration database Implementation aside
falken:
- (approx): Chrome does have a registration storage that data can either be
  directly stored in, or an external key (a la foreign key) can be stored.
asuth:
- I fess up we just key based on scope.

11:30p
### Removing Foreign Fetch fallout
#### link rel=serviceworker
Alex:
- Likes rel=serviceworker, but doesn't want it to be divergent in API.
bkelly:
- No plans to implement.  Would need to see compelling use-cases and adoption
  before we implemented it.
Jake:
- Wanted to be able to automatically add a SW for CDN's.
  - Would rather inject link rel than a script tag.
  - Header would be even easier.  (Alex: And less likely to disrupt CSP).
Alex:
- Developer interest is a good reason.
Jake:
- After cloudbleed, CDN's are scared about rewriting HTML and so would much
  prefer a header.

bkelly: a little worried
Alex: yeah, ServiceWorkers are dangerous.  and by using a CDN you're taking all
  the risks.

#### Install event
meta: issue about weird exception.
(wrong) marijn: we're still using extendable event.
bkelly: We're still using extendable event.
Jake:
- foreign fetch added methods onto install event.  Originally just extendable
  event.
- okay to go back to extendable event?
bkelly:
- Sure, it's fine.  But also fine to keep because it's got special logic.
falken:
- We've done it.
marijn:
- But we haven't done anything with it.  It's empty.

resolution: go back to ExtendableEvent.

#### What happens to an unconsumed fetch (POST) body?
https://github.com/w3c/ServiceWorker/issues/1191

everybody cool with JakeA's proposal.
- we mark the body instance (as reflected into the event for the request), we
  report the body as disturbed.
- and if the fetch handler did lock or otherwise disturb the body, a network
  error is generated.

bkelly:
- issue with rewinding the stream
falken:
- same deal, if a redirect happens, the body's already gone, basically.

Alex/Dominic:
- If you want to mess with the body, you need to explicitly tee/clone it.

### After-lunch
### Synthesized Redirect Stuff a la :bkelly
Complicated!  Complicated!  Oh, so complicated!

### rel=serviveworker CDN discussion
Webkit: yoav was ambivalent.  CDN's are already interacting with existing SW's,
so they already have to do sketchy re-writing.  So the feature doesn't help them
that much.

Alex: fast.ly does want header.  Because they don't ever want to touch/rewrite
content at all.

### worker scripts as sub-origin resources.
Complicated.

### async cookies API
n8s: Facebook interested in getting rid of cookie polling.

### 404 uninstall
consensus:
- No.  Facebook and Google infrastructure can return bogus 404's/etc., don't
  want SW's uninstalling because of that.
- Interventions are a possible option.
Alex:
- (approx)DNS is a rental service.  Web platform isn't really aware of this as
  of yet.  There are things that could be done.

### Range Requests
(don't let SW return a ranged response to something that wasn't a range request)

### Image Caching, raised by Ali
(spec what needs to be spec'ed)

### navigationType, issue 1167
Alex:
- Evaluated the various Google consumers, they all seem like they'd be fine with
  conjoined back_forward.

## new features

### dynamic addEventListener/removeEventListener for fetch
n8s:
- Use case of staleness for infrequently visited websites too.
Alex:
- Specifically tuned set of features for performance, not offline resilience.
- chrome looking at mitigations for ultra-slow SW startup.
### bypassing SW via attribute thing

### clients API on Window.
n8s:
- Sometimes page will postMessage to SW to get clients list on its behalf
  because interested in what else the SW can see.
bkelly:
- clients.open is nice over window.open because no "opener" and it's async.
- would like to expose SW as a client.
Jake:
- only worry about the multiple instances thing that we still don't know the
  shape of yet.
bkelly:
- But you need to use matchAll() with the right flags, etc.

### clientcontrolled / performance events...
NB: I lack full context on this block, no real xrefs provided.

Performance lifecycle stuff, going to dispatch to document, not the SW.  Might
expose states on the clients and then fire clientcontrolled events at the SW.

Jake:
- uncontrolled: Seems like bad idea to spin up a SW when you're out of
  resources.
Alex:
- This is a soft "get your house in order" thing.

### Recovering from fetch failures, issue 939
