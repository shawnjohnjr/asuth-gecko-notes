## Specific Errors to Report ##

* report detailed status on 404/friends in register()/update():
  https://bugzilla.mozilla.org/show_bug.cgi?id=1267473
* Bad MIME type: https://bugzilla.mozilla.org/show_bug.cgi?id=1233798
  * ServiceWorkerScriptCache.cpp CompareNetwork::OnStreamComplete checks MIME
    type (already main thread, simplifying)
* more info from throwing in different states, they exist, need (more) params
  and localization: https://bugzilla.mozilla.org/show_bug.cgi?id=1222720
  NB: currently assigned to bkelly


## Testing ##

There do not appear to be existing error tests for service worker.  The errors
seem important enough that tests are likely merited.  (There are however, some
tests of console.log and friends.)

In terms of verification, properly implemented error messages should be
localized using StringBundles by way of nsContentUtils::FormatLocalizedString
that builds on nsIStringBundle.FormatStringFromName.  So for testing, we
probably want to characterize the expected error as a dict of { bundle,
stringName, [ params ]}.  The expectation will be derived from the bundle.

### Specific Impl ###

* SimpleTest provides helpers: *need to check for async task helper*
  * monitorConsole/endMonitorConsole: monitorConsole takes strict ordering with
    explicit match logic (supporting regexps) which can include windowID.  Need
    to explicitly call
  * expectConsoleMessages(): wrapper around invoking a function and handling the
    monitor/end, optionally calling callback.
  * runTestExpectingConsoleMessages: wrapper that does the required
    waitForExplicitFinish and then directly chains into SimpleTest.finish.
    Only good if there's just the one test.

### Bugs ###


* Existing bug for existing tests / ReportToAllClients (presumed):
  https://bugzilla.mozilla.org/show_bug.cgi?id=1229156
  * Probably wants more detailed coverage of ReportToAllClients: have multiple
    windows at different level of scope match, plus variant with pending
    register?  Probably wants to be browser chrome test?
* Can the other per-error tests just use SpecialPowers?
