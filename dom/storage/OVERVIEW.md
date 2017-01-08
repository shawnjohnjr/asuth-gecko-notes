
### Participants

* DOMStorage: The representation all consumers deal with, 1:1 relationship with
  a specific global.  Internally holds a DOMStorageCache (many:1) which provides
  the raw storage.  Exposes:
  * "Storage.webidl" WebIDL binding to content.
  * nsIDOMStorage, the interface nsGlobalWindow consumes (plus the manager)

* DOMStorageCache: Actual (in-memory) storage, referenced by zero or more
  DOMStorage instances.  (Zero if being kept-alive in the cache.)  Uses
  DOMStorageDBBridge interface.
* DOMStorageCacheBridge: Abstract interface that DOMStorageCache exposes
  exclusively for its DOMStorageDBBridge to interact with.
* DOMStorageDBBridge: Abstract interface to either communicate directly with the
  DOMStorageDBThread in parent or indirectly via the DOMStorageDBChild IPC
  bridge mechanism in the child.
* DOMStorageDBThread: Actual database implementation of DOMStorageDBBridge.  Has
  (pre-e10s legacy) main-thread sync mReaderConnection for blocking reads, but
  (ideally) with most stuff happening via mWorkerConnection on its worker
  thread.  Extra stuff:
  * DBOperation: Database operation abstraction
  * PendingOperations: Smart DBOperation queueing with clever coalescing and
    mooting.
* DOMStorageDBChild

* DOMStorageObserver: Somewhat overrgown adapter layer from observer
  notifications to custom DOMStorageObserverSink idiom.  Has some business logic
  like delayed database read startup logic and transforms some browser topics
  to its own internal topics.
* DOMStorageManager: somehow concrete base class for its very shallow
  subclasses.
  * DOMLocalStorageManager: accessible as (singleton) service
    "@mozilla.org/dom/localStorage-manager;1", only one needed.
  * DOMSessionStorageManager: created per top-level-docshell (tab),
    corresponding to session storage's weird semantics.

### e10s


#### Loading

Requests come in as either:
* AsyncPreload: NewCache creates a CacheParentBridge, which is passed to
  db->AsyncPreload.  CacheParentBridge just knows how to create LoadRunnables
  that proxy from the db thread to the main thread to invoke SendLoadItem /
  SendLoadDone.  The CacheParentBridge will be released when the op is done and
  completes.
* Preload: A SyncLoadCacheHelper is created and passed to db->SyncPreload.
  SyncPreload knows to use LoadWait() which spins while the SyncLoadCacheHelper
  receives the LoadItem/LoadDone calls on the DB thread mutating the aKeys /
  aValues in place to be returned down to the caller.



#### IPC traffic ###





### Private browsing

There's some kind of issue with OriginAttributes where apparetly the API still
wants ~"isPrivate" passed in as a boolean.  Edge-cases seem to relate to the
system principal (which can never have the private browsing flag) and
"autostart" private-browsing.  See bug 1319951 for context.


### Startup, etc.

nsGlobalWindow::PreloadLocalStorage():
* hint function called at the end of nsGlobalWindow::SetNewDocument() to let
  the nsIDOMStorageManager->PrecacheStorage(principal) hopefully front-run any
  reads/mutations that would otherwise generate synchronous blocking.

DOMStorageManager::PrecacheStorage() calls GetStorageInternal() which:
* If the cache doesn't already exist, invokes
  DOMStorageDBBridge::ShouldPreloadOrigin to check if it should create a
  DOMStorageCache or not.  In the IPC case returns true if it doesn't know
  mOriginsHavingData yet, otherwise returns whether the origin is contained in
  the set.
* If it should preload, a DOMStorageCache is created a DOMStorageCache entry is
  created in mCaches and Init()ed.

*normal GetStorageInternal path*

DOMStorageCache::Init():
* Invokes Preload(), which calls DOMStorageDBBridge::AsyncPreload().
* Invokes DOMStorageManager:GetOriginUsage

DOMStorageManager::GetOriginUsage():
* Maybe early return if the usage is already known.
* Otherwise, DOMStorageDBBridge::AsyncGetUsage invoked, which will eventually
  result in the DOMStorageUsage::LoadUsage() being invoked which will increment
  the usage when received.

### Events / Notifications

DOMStorage generates observer notifications that are observed/consumed by
nsGlobalWindow.  In the event the window is frozen, the storage events are
accumulated in their own queue and re-run via explicit Observe calls as needed.

DOMStorage::SetItem and DOMStorage::RemoveItem both invoke
BroadcastChangeNotification if the respective DOMStorageCache call succeeded and
did not optimize the mutation into an NS_SUCCESS_DOM_NO_OPERATION rv.

DOMStorage::BroadcastChangeNotification creates a `StorageEvent` and creates and
dispatches a main-thread StorageNotifierRunnable to generate the observer
notification in a future turn.  The event strongly holds a reference to the
DOMStorage and the string key/oldValue/newValue.

The observer notifications:
* "dom-storage2-changed": non-private-browsing observer notification
* "dom-private-storage2-changed": private-browsing observer notification

nsGlobalWindow::Observe():
* Determines whether the event is relevant to the current window:
  * sessionStorage: Consistent with session storage's wacky per-tab semantics,
    asks the DOMStorageManager if the storage belongs to it.  (A principal /
    origin check would not be sufficiently scoped, and we seem to have evolved
    )
  * localStorage: Just checks principal equivalence.
* Checks whether the event in question is originating from this window itself
  (via check against mLocalStorage/mSessionStorage) because the spec calls for
  not emitting an event in that case.  But I guess for legacy reasons, we morph
  the event type to be MozSessionStorageChanged/MozLocalStorageChanged in those
  cases.
* nsGlobalWindow::CloneStorageEvent() creates a StorageEvent tied to the window
  global (the observer event used null).  Other than binding to the window, the
  only notable thing is that Get{Local,Session}Storage() are called which does
  super-duper matter.
* The cloned event either gets dispatched or queues in mPendingStorageEvents if
  IsFrozen().  (NB: It is probably indeed good that it's the clone that's
  getting stashed if frozen, even though the re-dispatch will clone again which
  is probably wasteful.  This avoids keeping another page's storage alive.)


### Blocking / Synchronous Reads ###

In the parent, either:
* when waiting for worker thread stuff, the mMonitor is used by
  DOMStorageCache::LoadWait() to block until DOMStorageCache::LoadDone() is
  called.
* DBOperations are run using mReaderStatements and stuff just happens
  synchronously.

In the child, the "sync" Preload IPC call is made to fill-in any gaps.  Because
of how actors/etc. work, there's no other way to do it, as the comment in the
implementation briefly covers.


### Database Writes

The database code is fairly clever and tries to only issue database writes every
5 seconds.  It observes this fairly strictly, the exception being that since
preload requests always hit the database, a preload request will trigger an
immediate flush if there are any changes pending for the origin.

### Misc

nsGlobalWindow::GetSessionStorage(), nsGlobalWindow::GetLocalStorage():
* perform access check and return if already available
* otherwise explicitly nsIDomStorage::CreateStorage
