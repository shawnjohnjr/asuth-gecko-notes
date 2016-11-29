## Test Stuff ##

Pre-change of skipWaiting behavior from "installed"=waiting state.

Useful context info:
- step_func() wraps functions in magic fail-the-test try/catch blocks.  It does
  not do any magic waiting for a promise or anything.  The one odd side-effect
  is that the function will be added to Test.steps, keeping the closure alive
  longer than it might otherwise be.

### Tests ###

service-worker/activation.https.html:

- "skipWaiting bypasses no controllee requirement" case just depends on
  mint-new-worker.py's "skip-waiting" flag to let the worker progress to
  activating, but it does not assert on it.

service-worker/skip-waiting-installed.https.html (*problem*):
- helpers:
  - local "onmessage" not-yet-active message handler which will resolve local
    "saw_message": listens for PASS (from skip-waiting-installed-worker.js)
    which asserts that skip-waiting-installed-worker.js is the active one on the
    registration and resolves.  *INCORRECT*/race-prone.
  - local "oncontrollerchanged" not-yet-active controllerchange handler which
    will resolve local "saw_controllerchanged" promise: asserts controller is
    skip-waiting-installed-worker.js (which is fine based on spec).
- setup: empty.js active, controlled iframe holding it active.
- controllerchange event setup on iframe, register of
  skip-waiting-installed-worker.js triggered, should result in "installing"
  slot.
- waits for installing SW to be installed=waiting.
- Hooks up "onmessage" to hacky MessageChannel event.source-workaround,
  postMessage()s that will cause the SW to invoke skipWaiting and post back.
  Waits on both saw_message and saw_controllerchanged.

service-worker/skip-waiting-using-registration.https.html (fine):
- helpers:
  - local oncontrollerchanged w/promise saw_controllerchanged.  asserts
    controller is "activating" and url2=skip-waiting-worker, resolves.
- setup: empty.js active, controlled iframe holding it active
- asserts controller, hooks up controller, registers url2=skip-waiting-worker
  which should result in url2 as "installing" worker.
- saves off registration, waits on controllerchange (which waits for it to be
  activating).
- asserts registration active, retrieves the 8 skipWaiting assert_equals checks
  from the worker via fetch_tests_from_worker().  (Completementary to
  skip-waiting-without-client.https.html which does it without a client.)

service-worker/skip-waiting-without-client.https.html (fine):
- uses resources/test-helpers.sub.js's service_worker_test to spin up
  skip-waiting-worker.js in a SW without an attached client.
  (skip-waiting-using-registration does it with a client.)

service-worker/skip-waiting-without-using-registration.https.html (fine):
- ensure no SW on scope
- spin up (uncontrolled) iframe on scope
- assert no controller, trigger register of skip-waiting-worker
- wait for activation (should not happen without skipwaiting)
- assert still no controller on the iframe (did not claim!), registration active
  pull the tests from the worker (like above 2).

service-worker/skip-waiting.https.html (fine):
- (setup for context below)
- Triggers register on skip-waiting-worker with empty.js (completely empty)
  active, and empty-worker.js (has a comment, but the differing file name is
  really all that matters) as waiting, with an inframe on the scope holding
  the document controlled.
- Waits for the freshly installing worker to progress to activated, which is
  sufficient guard.
- (The skip-waiting-worker.js internal use of the test infrastructure ends up
  not being used at all; it's just used for the side-effect of advancing the
  SW to activated.)

### Resources ###

service-worker/resources/skip-waiting-installed-worker.js:
- standalone; no use of testharness.js
- ('install' event handler as guard on 'message' event that worker is already
  installed.)
- 'message' event handler (with installed assert) invokes skipWaiting and
  postMessage()s back PASS when skipWaiting has resolved (with undefined).
  Note that it's assumed event.source and its postMessage is not available, so a
  MessagePort is actually transferred across to respond on.

service-worker/resources/skip-waiting-worker.js:
- uses worker_testharness.js and thereby tesharness.js.
- Invokes skipWaiting() 8 times inside a promise_test which is triggered by the
  "install" event.
  - The test ends when all the skipWaiting() promises have been resolved (with
    undefined, checked).
