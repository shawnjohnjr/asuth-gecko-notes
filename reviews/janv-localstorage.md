## Meta
Pref: dom.storage.next_gen
- true in nightly by ifdef
- false in rest.

### Disk changes:
- QuotaManager:
  - "ls" client:
    - "data.sqlite" SQLite database.  2 tables, default 1k pages size.
      - Minimal foo=bar, bar=baz database takes 8k.  Why?  Pages:
        - 1: sqlite_master
        - 2: ptrmap pages for incremental_vacuum (implicitly, need to check)
        - 3: database table
        - 4: data table
        - 5: sqlite_autoindex_data_1 for table data
        - 6: unknown...
        - 7..8: uh... minimum append size maybe?
  - in root: "ls-archive.sqlite"
- Original "webappsstore.sqlite" still gets data updated too.


```
CREATE TABLE database( origin TEXT NOT NULL, last_vacuum_time INTEGER NOT NULL DEFAULT 0, last_analyze_time INTEGER NOT NULL DEFAULT 0, last_vacuum_size INTEGER NOT NULL DEFAULT 0);
CREATE TABLE data( key TEXT PRIMARY KEY, value TEXT NOT NULL, compressed INTEGER NOT NULL DEFAULT 0, lastAccessTime INTEGER NOT NULL DEFAULT 0);
```

## New Review

### Oddities to follow up on:
- Client::TypeMax() replacing TYPE_MAX.

### Tests that we need to ensure exist:
- Creation of ls-archive.sqlite even if there was no webappsstore.sqlite.


## Previous Review

=== Part 1

Infra for PushEventQueue, requested additional commment.

=== Part 2

Straightforward on-right-thread helper.

=== Part 3

Includes the OnChannelReceivedMessage interrupt.

- Datastore is shared object.
- Database is actor for communication.
- If multiple frames, each frame gets its own actor.

- Currently will purge origins on PBackground. (because previously we'd do that
  on the I/O thread.)

- New implementation only caps 5 megs for each origin, rather than also
  enforcing group limit.

- Tests disabled that didn't pass, added back as the other patches fix them.

- LSObject is the binding
- LSDatabase communicates

Sorta discussed:
- PBackgroundLSDatabase uint64_t datastoreId: could be UUID

=== Part 4

QM assumes quotaobjects and directorylocks are freed by the time
ShutdownWorkThreads returns.

IDB has/had bug with OpenDatabase operations being in flight; they have their
own OpenDirectory lock and can have QuotaObject operations.  For LS, it's
PrepareDatastore that corresponds to OpenDatabase.  There must be no
PrepareDatastore operations, no Datastores, and no live databases.

Created 2-step closing for database actors.  Normally child just sends close
message to the parent, and if it's the last database actor, then the datastore
is also destroyed.  In some cases, such as shutdown, we have to close things
from the parent side.  To avoid races, it's just a request.

ShutdownWorkThreads will get more complex in the next patches.

=== Part 5

Real files on disk
Real directory locks.
Infrastructure for database keying.

Database actors now explicitly tracked.  Datastore didn't know about Databases
before.

AbortOperations, for when we get ClearStoragesForPrincipal.  We request an
exclusive lock, but we also abort operations.  Because we know the stuff is
going to be deleted, so why wait for existing operations to finish.

DirectoryLocks introduced, must be released at the right time.

=== Part 6

=== Part 8

Introduces connection.

5-second delay.

PrepareDatastoreOp::CheckClosingDatastore() handles the case where a datastore
is in the process of closing, but hasn't closed yet.  Lets it chain on.

May be in subsequent patch, if we get colliding datastore prepare operations,
only the first one runs, the rest just wait for it.  They just check the
hashtable.

=== privatebrowsing

=== storage event

removal of the apply logic moots the ability to flip implementation by pref.

Has explicit handling of the observer mechanism.

Only one observer per parent process.

XXX aPrivateBrowsingId should be dropped in favor of using the one from the
origin attributes.

TODO:
For me to look into: there's an optimization so that events for the same content
process aren't round-tripped.  Datastore::NotifyObservers doesn't send the event
back down.  So we do expect event ordering to potentially be skewed from the
ground truth sequencing that results from the sync IPCs, but the fact that the
events describe the change means that content that cares can optimize.

=== remove session only

This just removes the tests for the weird semantics.

TODO: Make sure we do IsStorageAllowed and cause session-only to be
non-persistent.  Right now, the patches don't address that.

=== honor storage pref

uses CanUseStorage, but ignores mIsSessionOnly, TODO needs that verified.

=== low disk space mode

=== client only storage mode (wipe out a single QuotaClient)

improve baku's patch from bug 1402254

=== fix removeLocalStorage in webextensions

fixes fallout from extension:purge-localstorage observer (effective) removal

=== new LocalStorageManager

also fixes forget about site

cleans up various ugly test observer things; some of which overlaps with
StorageActivityService.

=== fix intermittently failing test

Just a verify fix to remove the database when done.

=== Fix test_bug600307-DBOps.html

Adds testing-only (via pref) reload() method.

(may need to fix preferences clearing mechanism if not fully fixed in all
ForgetAboutSite stuff.)

=== e10s test changes, primarily preloading enhancements

The plan is that ContentParent will trigger the preloading via XPCOM
w/keepalive.

=== quota checks
