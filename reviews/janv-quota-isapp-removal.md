https://bugzilla.mozilla.org/show_bug.cgi?id=1311057

## 2017/01/30 new Part 4

head.js:
* much of this is derived from IDB's unit/xpcshell-head-parent-process.js

QuotaManager{Service}::Init
* Just starts up QuotaManager if it wasn't already.  Basically a NOP that
  requires initialization and asserts things are initialized when it runs,
  which provides an ordering gurantee when the callback fires.

## 2017/01/15

Most of this is just rote removal.  There are some non-obvious changes.

### Non-obvious things to track. ###

* QuotaManager::InitializeRepository: Losing the IsTreatedAsPersistent
  fast-continue which causes us to skip the call to InitializeOrigin
  * callee context: called by EnsureOriginIsInitialized, which is used by
    DoDirectoryWork when mGetGroupUsage is invoked.  It gets called with
    PERSISTENCE_TYPE_TEMPORARY.  The calls to InitializeRepository are made
    because !mTemporaryStorageInitialized; they're made on
    PERSISTENCE_TYPE_TEMPORARY and the ComplementaryPersistenceType,
    PERSISTENCE_TYPE_DEFAULT.
  * caller context: calls InitializeOrigin which is about usage tracking.
  * Understanding: This is all about counting up the temporary/default storages,
    without isApp, there's no longer any magic upconversion in
    IsTreatedAsPersistent; the persistence type is just the persistence type.
    We may expect to see a code path that would have persistent storage scans
    reach into default to look for things to upconvert in cases where we care
    about persistent storage.  (In general we don't for quota purposes.  But
    maybe we have tally logic somewhere for UI purposes or testing?)

* QuotaManager::IsQuotaEnforced no longer takes origin and isApp.
  * Straighforward-ish: the origin was unused and the isapp only mattered
    because we only care if the persistence is temporary/non-persistent.

Previously: IsFirstPromptRequired === !temporary && !whitelisted(origin)

Revised FactoryOp::CheckPermission:
* Now does check if persistence==PERSISTENT && !whitelisted.
* before: if(isFirstPromptRequired), so !temporary && !whitelisted, which is
  equivalent to persistent && !whitelisted.

For AsmJSCache:
* mEnforcingQuota was already always true; both the initial value and the
  use of IsQuotaEnforced that would always return true.
*
