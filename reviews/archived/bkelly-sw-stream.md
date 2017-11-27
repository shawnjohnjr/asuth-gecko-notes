https://bugzilla.mozilla.org/show_bug.cgi?id=1204254

## Start / Finish Event Ordering ##

Only relevant issue is start/finish runnable ordering.  Part 4 moves the start
runnable from using the worker's thread queue (which is subject to throttling)
to the system group.  This avoids the FinishResponse runnable being run prior
to the StartResponse runnable.

### Fundamental Underlying Issue ###
The concern is that the NS_AsyncCopy is happening on the
STS
