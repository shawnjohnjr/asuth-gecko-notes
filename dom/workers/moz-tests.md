## Tests ##

Testing notes:
* Pref "dom.serviceWorkers.testing.enabled" which most tests set causes all
  origins to be "authenticated"/secure as far as ServiceWorkerManager is
  concerned.

### Alphabetical ###
(So I can know what I've checked and categorized.  Note that there are
intentionally gaps in here right now for time/delayed pragmatism reasons.)

uncategorized:
* test_install_event: register, register different thing with same scope and get
  the same registration back, register an SW that will error in install and
  become "redundant", verify previous registration held; register worker that
  will install but error during activation, verify worker still activates
  despite error.

categorized:
...
* test_notificationclick_focus: client.focus() works on notificationclick event
  in timely fashion (100ms < 1000ms) and fails when oo late (2000ms > 1000ms).
* test_notificationclick-otherwindow: SW gets notificationclick event triggered
  using about:blank sub-iframe's prototype chain (but still same registration,
  seems sketchy)
* test_notificationclick: SW gets notificationclick event with complex data
  object intact.
* test_notificationclose: SW gets notificationclose event with complex data
  object intact.
* test_opaque_intercept: SW intercepts a script load, returning cached followed
  redirect that is therefore opaque, but has side effect when loaded.
* test_openWindow: Tests client.openWindow for various URLs and with popups
  allowed due to having clicked on a notification (magic via pref).
* test_origin_after_redirect*: Check documents opened via intercept that
  returned (possibly cached) opaqueredirect (so they return the un-pierced
  redirect) perceive the correct document.domain and location.origin.
* test_post_message_advanced: Basically checks postMessage is using structured
  clone by passing primitive types and Blob and File.
* test_post_message_source: messaged installing SW can use event.source to
  postMessage back to registering doc.
* test_post_message: Uncontrolled doc can postMessage the SW via the
  registration, SW can matchAll() and postMessage the controlled doc.
* test_privateBrowsing: make sure that if we load a controlled URL in a PB
  window that the doc is not controlled.
* test_register_base: Using a "base" tag with a secure origin doesn't trick
  register into thinking the document it secure.  (The magic pref that makes all
  origins secure is not used.)
* test_register_https_in_http: Can't register a serviceworker in a secure iframe
  embedded in an insecure iframe.
* test_request_context/test_request_context_FOO: Check that the serviceworker
  sees the correct request.context values when intercepting.
* test_sandbox_intercept: SPEC-CONTRADICTING test to ensure sandbox iframes are
  never intercepted.
* test_sanitize_domain: Use SpecialPowers to emit browser:purge-domain-data for
  "example.com" verifying it nukes only the given registration and not others.
* test_sanitize: Use SpecialPowers to emit browser:purge-session-history
  observer notification and verify it nukes the registration made.
* test_scopes: Register a worker with a bunch of different scopes; verify glob
  logic using testing-only getScopeForUrl helper.
* test_service_worker_allowed: Check that Service-Worker-Allowed header which
  limits scope is enforced.
* test_serviceworker_header: Check that "Service-Worker: script" header is
  present
* test_serviceworker_interfaces: check exposed ECMA globals and DOM interfaces,
  some of which are contingent based on build type.
* test_serviceworker_not_sharedworker: ServiceWorker doesn't get unified with
  SharedWorker; create SW and SharedWorker with same URL and scope, make sure
  they're different things.
* test_serviceworkerinfo.xul: Test SWM identity invariants and impact of
  debugger (attaching restarts, detaching eventually triggers shutdown)
* test_serviceworkermanager.xul: Checks notification rules (only when scopes
  are registered/unregistered, not changed).
* test_serviceworkerregistrationinfo.xul: Check registration notifies of changes
  and the progression of installingWorker, installingWorker.scriptSpec, waitingWorker, activeWorker
* test_skip_waiting: skipWaiting in SW (from existing controlled scope) works
* test_strict_mode_warning: Registration should still pass even if there are
  strict mode warnings in the SW file.
* test_third_party_iframes: Test impact of network.cookie.cookieBehavior on
  interception of third-party origins; COOKIE_BEHAVIOR_REJECTFOREIGN should
  disable interception.
* test_unregister: unregister from a document, spinning up a new iframe under
  the scope to verify uncontrolled
* test_unresolved_fetch_interception: Unresolved respondWith channel gets reset
  when SW is terminated for idling.
* test_workerUnregister: test SW unregistering itself when poked by message from
  a controlled document.
* test_workerUpdate: explicit update from the service worker itself
* test_workerupdatefoundevent: issue a second register to trigger an updatefound
  event on the registration inside the worker
* test_xslt: XSLT interception

### Categorized ###

* Chrome-privileged stuff
  * ServiceWorkerManager
    * Test SWM identity invariants and impact of debugger (attaching restarts,
      detaching eventually triggers shutdown): test_serviceworkerinfo.xul
    * Notifications of scope register/unregister: test_serviceworkermanager.xul
    * Check registration notifies of changes and the progression of
      installingWorker, installingWorker.scriptSpec, waitingWorker,
      activeWorker: test_serviceworkerregistrationinfo.xul
  * SpecialPowers approximating user-actions/chrome-level stuff
    * Use SpecialPowers to emit browser:purge-session-history observer
      notification and verify it nukes all registrations by checking a
      single registration: test_sanitize
    * Use SpecialPowers to emit browser:purge-domain-data for "example.com"
      verifying it nukes only the given registration and not others:
      test_sanitize_domain
* Client interaction
  * focus
    * client.focus() works on notificationclick event in timely fashion (100ms <
      1000ms) and fails when oo late (2000ms > 1000ms):
      test_notificationclick_focus
  * openWindow
    * Tests client.openWindow for various URLs and with popups allowed due to
      having clicked on a notification (magic via pref): test_openWindow
  * postMessage
    * Uncontrolled doc can postMessage the SW via the registration, SW can
      matchAll() and postMessage the controlled doc: test_post_message
    * messaged installing SW can use event.source to postMessage back to
      registering doc: test_post_message_source
    * Basically checks postMessage is using structured clone by passing
      primitive types and Blob and File:test_post_message_advanced
* Interception
  * Private Browsing
    * make sure that if we load a controlled URL in a PB window that the doc is
      not controlled: test_privateBrowsing
  * Redirects
    * SW intercepts a script load, returning cached followed redirect that is
      therefore opaque, but has side effect when loaded: test_opaque_intercept
    * Check documents opened via intercept that returned (possibly cached)
      opaqueredirect (so they return the un-pierced redirect) perceive the
      correct document.domain and location.origin: test_origin_after_redirect*
  * Request
    * Check that the serviceworker sees the correct request.context values when
      intercepting: test_request_context/test_request_context_FOO
  * Sandbox
    * SPEC-CONTRADICTING test to ensure sandbox iframes are never intercepted:
      test_sandbox_intercept
  * Third-party
    * Varying based on network.cookie.cookieBehavior;
      COOKIE_BEHAVIOR_REJECTFOREIGN should disable interception:
      test_third_party_iframes
  * XSLT: test_xslt
* Notifications
  * Events:
    * click: SW gets notificationclick event with complex data object intact:
      test_notificationclick, variant test_notificationclick-otherwindow,
      variant test_notificationclick_focus
    * close: SW gets notificationclose event with complex data object intact:
      test_notificationclose
* Registration
  * Activation
    * skipWaiting in SW (from existing controlled scope) works:
      test_skip_waiting
  * Fetching:
    * Check that "Service-Worker: script" header is present:
      test_serviceworker_header
  * Problems with worker
    * Registration should still pass even if there are strict mode warnings in
      the SW file: test_string_mode_warning
  * Security / Limitations
    * Check that Service-Worker-Allowed header which limits scope is enforced:
      test_service_worker_allowed
    * Secure contexts
      * Can't register a serviceworker in a secure iframe embedded in an
        insecure iframe: test_register_https_in_http
      * Using a "base" tag with a secure origin doesn't trick register into
        thinking the document it secure.  (The magic pref that makes all origins
        secure is not used.)
  * Scopes
    * Register a worker with a bunch of different scopes; verify glob logic
      using testing-only getScopeForUrl helper: test_scopes
* Termination
  * Unresolved respondWith channel gets reset when SW is terminated for idling:
    test_unresolved_fetch_interception
* Updating
  * Explicit update from inside worker: test_workerUpdate
  * Second overlapping register triggers updatefound event on the registration
    inside the worker.
* Unregistering
  * document unregistering SW and verifying with new doc: test_unregister
  * SW unregistering itself when poked by a message from controlled document:
    test_workerUnregister
* Worked fundamentals/-related
  * Globals
    * check exposed ECMA globals and DOM interfaces, some of which are
      contingent based on build type: test_serviceworker_interfaces
  * Not other things
    * ServiceWorker doesn't get unified with SharedWorker; create SW and
      SharedWorker with same URL and scope, make sure they're different things:
      test_serviceworker_not_sharedworker
