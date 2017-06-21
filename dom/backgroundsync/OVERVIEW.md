Much of the implementation and structure is cribbed from dom/cache.

The important moving pieces:
* StorageManager: per-origin state manager.  Uses the now-shared ClientContext
  support infra to deal with the QuotaManager issues.

### Database Storage ###

One-off:
* PROFILE/backgroundsync_chrome.sqlite:

Per-origin:
* backgroundsync/backgroundsync.sqlite

### Threads ###

* "BsyncChIOThread": 1 singleton
* "BSyncIOThread":  1 per StorageManager (per origin)

### Testing ###

https://deanhume.github.io/Service-Workers-BackgroundSync/
