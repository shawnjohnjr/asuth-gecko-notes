# SPIDERMONKEY_PROMISE #

https://bugzilla.mozilla.org/show_bug.cgi?id=911216 is trying to move the
promises implementation into spidermonkey from DOM.  SPIDERMONKEY_PROMISE is
the big switch.  The DOM changes were on
https://bugzilla.mozilla.org/show_bug.cgi?id=1243001 and have already landed
and the comment 0 is great.

The big take-aways here are:
* dom::Promise is not wrapper-cached and

## On WorkerHolders ##

My operating theory right now based on code inspection is that a
PromiseNativeHandler on a worker will not automatically be resolved/rejected in
the event of worker death.  Instead, it will just lose the relevant refcount(s).

This does mean that a WorkerHolder is necessary if the promise handler is owned
by anything that can outlive the loss of the worker (possibly via mutual cycle).

# Existing DOM promises #

## PromisedWorkerProxy ##
