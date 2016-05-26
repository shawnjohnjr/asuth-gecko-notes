# SPIDERMONKEY_PROMISE #

https://bugzilla.mozilla.org/show_bug.cgi?id=911216 is trying to move the
promises implementation into spidermonkey from DOM.  SPIDERMONKEY_PROMISE is
the big switch.  The DOM changes were on
https://bugzilla.mozilla.org/show_bug.cgi?id=1243001 and have already landed
and the comment 0 is great.

The big take-aways here are:
* dom::Promise is not wrapper-cached and

# Existing DOM promises #

## PromisedWorkerProxy ##
