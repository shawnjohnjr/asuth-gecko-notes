## Known In-Tree Consumers ##

### Sync API Users ###

* DOMStorage (aka LocalStorage): dom/storage/DOMStorageDBThread
  * 2 synchronous connections:
    * mWorkerConnection: sync conn opened and used on its own "worker" thread
    * mReaderConnection: sync conn cloned from mWorkerConnection by the worker
      connection when it initializes.
  * Database abstraction uses operations which may run on the main thread or
    worker thread, picking the connection based on what thread they're on.  (The
    local storage read when in the same process may block.)
* IndexedDB
* Places / Core: Still uses some kind of hybrid
* Satchel / Form history, deprecated possibly dead synchronous nsIFormHistory2 implementation at toolkit/components/satchel/nsFormHistory.js
  * https://bugzilla.mozilla.org/show_bug.cgi?id=876002 tracks removal.
  * There is a more modern, used async consumer in FormHistory.jsm


### Async API Users ###

* Places / Core: Weird hybrid stuff.
* Places / ConcurrentStatementsHolder
  * Performs an AsyncClone on the Places DB's
* Places / nsPlacesAutoComplete.js
* Satchel / Form history: toolkit/components/satchel/FormHistory.jsm
  * Note: has deprecated sibling synchronous-consumer nsFormHistory.js
  * Opens synchronously on the main thread and does initialization/migration
    there.
* Sqlite.jsm ()
* toolkit/mozapps/extensions/internal/AddonRepository_SQLiteMigrator.jsm
  * One-off migration logic from SQLite database to JSON.  Closes itself once
    it has retrieved the data.

## Shutdown Behavior ##

### Shutdown handled ###


* DOMStorage: Synchronous shutdown at "profile-before-change" (also
  "xpcom-shutdown").  The main thread notifies the "worker" and joins on the
  thread which does a synchronous Close().
* IndexedDB: QuotaManager observes "profile-before-change", dispatching a
  runnable to PBackground which does the real shutdown, cascading through to
  IndexedDB's connection pool shutdown which spins until all its connections
  are closed/etc.
* Places / Database: Registers with nsIAsyncShutdownClient with database
  shutdown blocker added to the "profile-before-change".  Triggers AsyncClose
  with the callback releasing the blocker.
  * Adds a blocker for client shutdown to "profile-change-teardown" phase.
  * Also converts "profile-change-teardown" into "places-shutdown" observer
    notifications with its own observer.
* Places / ConcurrentStatementsHolder: Issues *blind* AsyncClose on its cloned
  read-only connection when shutdown by History::Shutdown which observes
  "places-shutdown" (which Database notifies on "profile-change-teardown").
* Places / nsPlacesAutoComplete.js: Issues *blind* asyncClose on
  "places-shutdown" observer notification (which Database notifies on "profile-change-teardown").
* Satchel / Form History (FormHistory.jsm): Async shutdown triggered on
  "profile-before-change" that spins its own event loop.
* Sqlite.jsm: Adds an AsyncShutdown.profileBeforeChange blocker, so
  "profile-before-change".

### Oblivious to shutdown ###

* toolkit/mozapps/extensions/internal/AddonRepository_SQLiteMigrator.jsm just
  closes itself once it finishes the migration.
