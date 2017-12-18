# Actual Notes

## "Persistence" changes

1. Deprecate the persistent type, logging a warning, but still working.
2. Migrate the database to be default/temporary, rename it if there's a
   collision.

## Preferences, Privacy and Security.

eSessionOnly means clear-on-shutdown.

## Devtools serviceworkers

:salva interested in being able to tell the upstream source of respondWith as
displayed in network panel.  If we could show it came from a dynamic stream
versus a specific Cache.  (AKA what Cache matched?)


# Aborted bug comment notes to consider
For bug 1424459 I had the following thoughts that are too rambly.

I agree there's a loss of fidelity, but it's worth pointing out that that we currently have 2 other values here that are very special and I think don't want to be exposing the StorageAccess values as-is.

The decision for each implementation I think is:
a) Forbid use of the API entirely.
b) Allow use of the API and use an in-memory implementation that never touches disk (except as swap).
c) Allow use of the API and use a QM-managed disk-backed implementation (which may in turn be purged at shutdown depending on yet other preferences).



  I think we want it to be the case that storage is either explicitly allowed or explicitly denied

## ePrivateBrowsing ##

We are moving to pulling this explicitly off the origin attributes.  Other than the edge-case about the system principal and the risk of chrome-principaled code depending on the pre-originattribute approach to not do privacy-breaking things, this is obviously the right way to do it.

## eSessionScoped ##

This is tricky and ad-hoc.  We have longstanding implementations to support our non-standard prefs/permissions in our cookies and LocalStorage implementation.  We also made arguably the wrong call and allowed IndexedDB and the DOM Cache API to persist data to disk in these modes without wiping.

(You filed bug 1400678 that's related to this, although that's "Clear history when Firefox closes" which is the "privacy.sanitize.sanitizeOnShutdown" pref rather than the ["Accept cookies from websites...", "Keep Until...", "I close Firefox"] setting's "network.cookie.lifetimePolicy"=2 pref which globally sets the default that leads to eSessionScoped.  Until such time as we have a private-browsing implementation for IDB and Cache API that don't touch disk, however, the solution for clear-at-close is the same for session-only if we allow the APIs to work at all.)

I think there are really 2 ways to handle this mode:
1) Treat session-only mode as a special in-memory mode for only cookies and localstorage for legacy reasons, disable fancier storage like IDB and Cache API.
2) Make it an origin attribute and allow QM persistence that gets wiped at clean shutdown and again if it's found at startup to cover un-clean shutdown.  If we don't trust an origin to persist data to disk, should it be allowed to see other origins that might be able to persist things to disk?  This would also have nice characteristics for QuotaManager purposes since the init sweep could see the origin attribute says the storage needs to be nuked.
