### Edge

Background push SW won't see any clients at all right now.  That's an
implementation limitation, they can change that later.

In discussion of having the background process relay the event to the browser
process, Ali indicated concern over non-idempotent operations being triggered
multiple times.  (This is a weird space where the spec and reality demand and
recognize that there will be failures, but practically speaking, it is indeed
likely that content logic is likely to not implement their handlers in an
idempotent fashion and that increases in edge case incidence, as could happen if
the browser process gets clobbered by UI shutdown and requires the event to be
re-run in the background, will result in glitches.)

They're leaning towards having backgroundsync use the background process as well
because of that above scenario.  They can have the background SW continue to
live even if the UI/browser process is terminated.

No current plans for SharedWorker.

Foreign fetch under consideration.  

### Chrome

For their service-ification, planning to send both fetch events and postMessage
events directly to SW, bypassing their "central browser process".  Both fetch
and postMessage to ensure a consistent ordering.

(For other events, page can't observe ordering changes.)
