## Multiple SW Instances
Edge changed from background (potentially duplicate worker) push notification
shipped in April to forwarding the event in October.

Safari moving towards process-per-site.

Chrome evaluating a model where they can "migrate" a SW by first evaluating up
a SW in a new process while there's still a live one in another process, then
transferring delivery of fetch events once it's up.

## Resurrection
(Chrome screws up on resurrection, will definitely removed.)

## User presentation: LinkedIn presentation
- Kill-switch as a no-op that just installs and claims with skipWaiting().
  - Replaced existing SW after a beacon was fired on a fetch.  Looked to make
    sure events stopped after 10 minutes.  Users still firing after 10 minutes
    are marked as bad.
  - Test on 3.7 million devices, ~600 and 200 failures.
- Clear-site-data triggering a refresh is too much; don't want to interrupt
  users.
- Life-cycle concerns about eliminating a buggy SW.
- Propose a "soft terminate", terminates on earliest of:
  - 1 minute
  - full page load
  - all tasks complete.
- Pretty much a similar breakdown along all browsers of Chrome, Firefox, Edge.

Facebook has wrapper code that looks for special headers in every fetch request.
- Removes the event listener for the fetch event.  (We only limit
  addEventListener).

Both Facebook and LinkedIn are wrapping all their waitUntil() cases so that they
have a timeout or rather "deadline".

Also have timeout issues on push messages where they do a network request and
it may timeout for whatever reason, but they are required to show a desktop
notification or the browser penalizes them (and shows a fallback notification).

LinkedIn is using the Cache API for data storage preferred over IndexedDB due
to bad latency problems they've seen.

Facebook hasn't had problems with IDB but they only use it for logs.

Facebook still wants to bypass the ServiceWorker from the page when they know
what's happening.

## Performance Discussion

Ours:
- Streams coming
- Cache API alternate data stream.
- IndexedDB performance

### Google
Google Search started using SW at google.com/search.
Google search forbids bfcache on back for revenue reasons.  They need client
side logic to make it happen.

google.com/maps?force=pwa

Google has question of who owns the root.  Google wants the root SW to
explicitly disclaim control of sub-dirs.

### Static Routing
Explicitly only about things you could write yourself in a "fetch" event
handler.  Not about registration matching.

## Cache Eviction
Chrome does a clever thing with alternate data stream keying secondarily on the
URL.

We've discussed putting Request and Response objects into IndexedDB. :bkelly is
worried about it being a serious optimization problem given that transferring
into IndexedDB would need to consume the stream.
