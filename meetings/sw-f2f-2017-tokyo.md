## Monday ##

###

No changes in Chrome clients API implementation of late.

Ali/MS: Public ServiceWorker release neutered, it's behind a flag and a no-op.
Windows insider will have working ServiceWorker.  Caches behind a flag in
creator update; no quota management which is why behind a flag.  They're
re-doing quota management.  Planning to integrate with OS background task system
for push notification handling, similar plan for background sync, but that's
further off.  One service worker per process per domain?.  Devtools will split
out extra window for serviceworker.  F12 instances are separate worlds and
there's some devtools network tab limitations.  MS store slurping bing data and
filtered web manifests to expose as "HWA" Hosted Web app.  HWA's have separate
storage (with separate, unbounded limits).  Will have ability to claim the
entry.  Do have FetcyBodyStream, but not general ReadableStreams.  For next
release, SW's may still be behind flag depending on status, but will actually do
something.

Falken/Google/Chrome: Focusing on performance.
* Navigation preload implemented, origin trial still happening.
* Tried speculative SW startup (ex: based on mousedown), experiment canceled.
* Working on: avoiding main thread for fetches from SW, and more efficient
  streaming from SW to page.
* Planned soon: Startup SW thread off the main thread, right now the main thread
  needs to initiate it.  Reduce roundtrips through the privileged browser
  process from un-privileged render process. (Like communications between SW
  and page.)
* Actively moving chrome tests to WPT and working on passing the WPT's.
* Did a foreign fetch origin trial, analyzing.
* Bugfix: avoid allowing abusive tricks to keep SW alive forever.
* SW's can now have their UseCounter telemetry.
* Implemented Response#redirected and CSP/SRI ("security") restrictions.
* Working on useCache/"no-cache" by default
* Working on Background fetch.
* Planned soon: byte-for-byte comparison for importScripts()
* Primarily focused on performance.
* readable and writable streams.  decoding streams behind a pref.

Facebook: Goal is to speed up by mixing the initial parts of the page from cache
with fresh data (via preload).  Tricks:
* Priming the response by returning an empty readablestream games Chrome's
  strategy that waits for something from the server.
* Stash things in global state over Cache.
* Cloning streams bad for performance.
* analyzing by CPU cores; SW only got better (on Windows 7 on Chrome) for 4 and
  8 cores.  2 cores was slower.  And this is with active SW, no startup cost.
* overhead of SW intercepting things that it's known won't be handled is bad
  with videos or other media or content.
* Long term test without navigation preload; it's an improvement when there's no
  SW startup wait, it's about even with startup.  Origin test shows ~150ms
  improvement on beta/canary which is actually correlated with slowness because
  a lot of extensions/plugins or something like that.  Hoping it carries through
  on release.
* Tests have not involve an appshell.  Facebook has an "earlyflush" concept for
  non-content stuff that gets sent immediately.  The SW stuff was just
  optimizing earlyflush.
* Working on appshell approach, but it adds complexity for version mismatches.
  Previously could just redirect on mismatch?  They do an 80%/20% rollout
  sometimes... working on teaching servers about new/old versions.
* Doing a lot of work with reusing data between installing/activating, having
  different caches but also share/reuse.  Also, currently, installing SW doesn't
  realize it's installing immediately.  So want a way to know from within the SW
  to break symmetry.
* Want to launch with SW caching in Chrome in the next 2 months.
* bytecode cache important.

Discussion of compilation/optimization.  Ability to optimize Cache on put() in
installing SW's.  Have the put() wait for optimize because installing is not
time sensitive and people want to know things are optimized.

### Responses and URL's

Response constructor won't let you pass in a URL.  But you can control what URL
the page requests and you can generate redirects.

## Tuesday ##

### Multiple Service Worker Instances ###

For normal SW's, would need to be opt-in.  For Foreign Fetch discussion of doing
it by default.

Edge is running pushes in background tasks in own process distinct from browser
process, Safari likes that.  Alex proposes simple hack for Edge where they make
sure there's never a SW alive in both places at the same time.

FB postMessage to propagate/reconcile state in addition to telling it to open
badges.  FB gets notification, posts a messages to a client saying navigate.
Client posts a message back saying "hey, I navigated."  Notification case an
SW sets up a MessageChannel, but they can see a case where it would happen the
other way.

### Foreign Fetch ###
Google fonts use-case involves dynamically creating the stylesheet on the fly.
(Tons of permutations.)

### bading ###

Safari, no interest in dynamic favicon changing,  but yes to badging.  Title is
fine as long as it's suggestive, not a requirement.  They categorically don't
support dynamic icon updates.
