# Landing Effort #

## Outstanding Action Items ##

* part 4.1 principal/origin passing https://bugzilla.mozilla.org/show_bug.cgi?id=1217544#c133
* Quota Client stuff
  * DONE Propagate bkelly changes per https://bugzilla.mozilla.org/show_bug.cgi?id=1217544#c125
    * first set 46cb4ee49786
      * Context.{h,cpp} was all that was needed
    * second set a522c83358fa
  * DONE Propagate :janv's bug 1311057 isApp quota manager changes, efb4ec26174e
  * DONE Propagate :janv's changes from https://bugzilla.mozilla.org/show_bug.cgi?id=1348660
  * NOOP: Part 5 comments from :ferjm:
    * This patch takes dom/cache/ Context* and Action* to dom/quota/, renaming them as ClientContext* and ClientAction* where "Client" refers to a QuotaManager client. Since that's in essence what they offer: doing all the QM initialization work so storage actions can happen.
    * The main change is the addition of a "Listener" interface that dom/cache/Manager and dom/backgroundSync/StorageManager must use.
* DONE: mozStorage things
  * my comments at https://bugzilla.mozilla.org/show_bug.cgi?id=1217544#c129
  * mak's comments at https://bugzilla.mozilla.org/show_bug.cgi?id=1217544#c130
* DOING: part 9.1 review comments https://bugzilla.mozilla.org/show_bug.cgi?id=1217544#c134
* DONE: Finish permission fixups consistent with https://bugzilla.mozilla.org/show_bug.cgi?id=1355608

* **wait for active before processing register** it's step 2.
* try server problems revealed:
  * the interfaces aren't exposed?  This happened everywhere.  I clearly screwed
    up somehow unless I've since fixed that?  Maybe the second pref?
  * race on post-firing getTags().
    * The specific problem is that the removal job has to freshly get a
      StorageManagerId via main-thread bounce.  This races the getTags call
      which already has a StorageManagerId for itself.   So getTags() gets run
      before the removal takes effect.
    * getTags() returns the registration until it's removed upon fulfillment.
      So really we want a test like: getTags(), waitUntilledResolve(), getTags()
      showing that the tag was present in the pre and disappeared in the post.
  * getSyncEvents() timeout because of process mismatch.  Need the process
    probing thing.  Suggest putting in ServiceWorkerManagerService.
  * Shutdown/cleanup problem.  The StorageManager and StorageManagerId instances
    were hanging around.  Sugg



## Additional To-Do's ##

* Expose debugging information on nsIServiceWorkerManager /
  nsIServiceWorkerRegistrationInfo.

* Multiple attempts handling, plan:
  * Persistently Track:
    * Number of attempts where the network was believed entirely online.
      Increment this value each time we dispatch the event.
    * Number of attempts where the network disappeared.  We flag the active
      sync as having been doomed in our in-memory status and if it fails, we
      subtract 1 from the onlineAttempts and add 1 to the doomedAttempts.
    * Last sync attempt, wall time.  Internally, we use this to tell if it's
      time yet to perform another sync attempt.  Externally, this is potentially
      useful for debugging.
  * In-memory tracking:
    *



## Additional Tests to add ##

* QuotaManager, removal of a specific origin or all origins should remove the
  origin from the ChromeStorage as well.  Need debug helper via XPCOM that lets
  us invoke and return the GetAll() request results.
* getTags sequencing() from above.  getTags(), waitUntilledResolve(), getTags().

## Extra Reviews Needed? ##

## Parts List and To-Do ##

* Part 1: Interface
* Part 2: ServiceWorkerRegistration
* Part 3: IPC
* Part 4: "sync" event
* Part 5: QuotaClient helpers **r? bkelly**
* Part 6: mozStorageConnectionUTils
* Part 7: Storage/Persistence **r? bkelly**
* Part 8: Online state observer
* Part 9: core logic: BackgroundSyncService
* Part 10: IPC, multi-origin, crash resiliency fixes

(prefs)
* Part 12: Basic Tests
* Part 13: Multi-origin tests

## Part 10 stashed notes ##

Bug 1217544 - Part 10: IPC, multi-origin, and crash resiliency fixes. r?=bkelly

IPC:
- Don't send the sync event to all child content processes.  Instead,
  pick a single ServiceWorkerManager to tell to dispatch the event.
  For registrations that immediately fire an event, use the SWM
  corresponding to the process that the request originated from.  For
  online transition cases, pick the first content process, creating one
  if e10s mode is enabled and one doesn't exist.
- Ensure permissions are available before potentially causing the
  ServiceWorkerPrivate to be spun up using the bug 1355608 mechanism.

multi-origin:
- GetStorageManagerRunnable did not handle the multi-origin case
  correctly.  It mutated its single mPrincipalInfo in a tight loop during
  which it dispatched itself to the main thread (multiple times) to
  consume mPrincipalInfo.  Depending on scheduling, seeing all the requests
  as involving the same origin would be possible.
- GetStorageManagerRunnable was also a bEventTargetit silly.  It would go to the main
  thread to convert an origin to a Principal and then into a PrincipalInfo.
  When it got back to the background thread, it would use
  StorageManagerIdFactory::Create which would then go to the main thread
  to convert the PrincipalInfo back into a Principal and actually create
  the StorageManagerId.
- Consistent with :bkelly's suggestion in
  https://bugzilla.mozilla.org/show_bug.cgi?id=1217544#c98,
  GetStorageManagerRunnable and StorageManagerIdFactory were replaced with
  a MozPromise based solution.  Further cleanup/normalization is possible
  but the diff was already getting larger than intended.

crash resiliency:
- Before dispatching the &quot;sync&quot; event, the BackgroundSyncService would
  change the state of the registration to &quot;firing&quot; on disk.  A &quot;firing&quot;
  registration would not fire again.  If the system went offline again
  before the state change returned, the state would be changed back to
  &quot;pending&quot;.  But no logic addressed the case where a crash occurred
  and at next startup the system came up with registrations already in
  the firing state.  The only way to recover from this state would be
  for user code to remove the existing sync and re-register.  Removing
  would succeed, but duplicative register() calls would fail because
  of primary key collisions.
- The &quot;fix&quot; for this, which also happened to simplify propagating the
  actor for the IPC process-affinity for firing the event, is to just
  maintain an in-memory map of all currently pending firing events.
  This is currently somewhat overkill since the simplification also
  allowed the entire dispatch to become synchronous with the removal
  happening later in the method.  However, when the follow-on
  bug 1260141 fix happens, the map is arguably the right way to handle
  that still, although it wants a little bit of fleshing out to deal
  with the reregisteredWhileFiring state.  As far as the disk state
  is concerned, all that matters is whether a sync should happen at
  next startup and any back-off or last chance factors.  The largest
  complication is that registration id generation currently only
  happens in the database layer which does complicate single source of
  truth notions.  However, that can be left as-is if the &quot;registration
  already exists&quot; case is propagated back to the PBackground thread
  with the actual current registration and its id, allowing the map
  to be properly fixed up or the request re-issued if the map no longer
  has an entry.  Alternately, the PBackground state can be made a
  more authoritative source of truth for situations like that by having
  it track pending requests issued to the managers until they
  complete.  While there's something to be said for the managers and IO
  thread being a single source of truth, it's massive overkill for
  this use-case where really a simple async key-value store is all that's
  needed.

## Telemetry Metrics Proposal ##

Need to block: ServiceWorkers-data alias, https://bugzilla.mozilla.org/show_bug.cgi?id=1328391

* Per-online transition:
  * Total number of sync events dispatched at online event.
  * Sync event
* Per sync registration:
  * Time between sync event registration and servicing.  0 for immediately
    dispatched events.  (Unless zero values can't happen?)

# Original #

## Database/Storage ##
Using backgroundsync file names since they're somewhat more split/renamed to be
more accurate.

Same (modulo smaller things that can be parameterized):
* StorageAction.h/cpp (from Action.h/cpp)
* DBAction: Opportunistic connection reuse, logic related to opening and reusing
  databases, possibly re-creating from schema.
* DBConnection (Connection in dom/cache): exists to wrap mozIStorageConnection
  in order to provide an automatic incremental vacuum on connection close.
  Otherwise manual boilerplate-y wrapper.
* DBSchemaUtils: InitializeConnection with WAL/page/growth parameters and sanity
  checks.  Genericized CreateOrMigrateSchema that takes SQL,
  currently-limited-to-one table-names.  IncrementalVacuum with sanity checks.
  Validate with DEBUG-only exact parameterization of expected tables and indices
  and SQL in cache case, which could probably be pulled out to be non-debug
  exact characterization of schema.  Somewhat altered in backgroundsync to take
  single parameter.
* QuotaClient: Recursive file-walker with some limited hard-coded filename
  prefixes.  dom/cache has extra directory check logic to handle existince of
  "morgue"

Domain specific
* DBSchema: Where all the actual specific database logic happens.

INVESTIGATE MORE, but cargo culted

* StorageContext.h/cpp (from Context.h/cpp)
