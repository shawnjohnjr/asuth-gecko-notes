
## dom/quota

ActorsParent.cpp:
- QuotaManager
  - MaybeCreateLocalStorageArchive(): creates PROFILE/storage/ls-archive.sqlite
    if it does not already exist by copying the contents of webappsstore.sqlite.
    - Opens the database to checkpoint the WAL.
    - Copies using a -tmp suffix.
    - Converts the database to rollback mode (via journal_mode = DELETE).
    - Has fallback that explicitly creates an empty database.

Client.h:
- Client
  - "LS" type added after SDB.
  - TypeMax() introduced that checked CachedNextGenLocalStorageEnabled() and
    returns TYPE_MAX if enabled and LS if not.
  - TypeToText() updated to be clever: Client::LS only mapped if enabled,
    otherwise it falls through to the unreachable.
  - Similarly string type/directory to type is conditional on being enabled.
  - NullableTypeFromText() introduced to let Nullable<Type> be used. *why?*


## dom/storage ##
LocalStorageManager.cpp:
- LocalStorageManager:
  - Now also implements nsILocalStorageManager but with the Preload(...) and
    IsPreloaded(...) checks stubbed out.
  - Constructor added NextGenLocalStorageEnabled() guard assertion.


## dom/localstorage ##
ActorsChild.h:

ActorsChild.cpp:

ActorsParent.h:

ActorsParent.cpp:

LocalStorageCommon.h:
- NextGenLocalStorageEnabled(): a main-thread-only public check of the pref that
  caches state.
  - Used by logic to gate:
    - nsGlobalWindowInner::GetLocalStorage chooses implementation based on this.
    - nsGlobalWindowInner::CloneStorageEvent() uses it to gate whether
      old-school hacky propagation is used or not.
    - nsGlobalWindowInner:::EventListenerAdded/EventListenerRemoved uses it to
      invoke EnsureObserver()/DropObserver() on the LSObject (if non-null).
      In the add case, this is after GetLocalStorage() which should bring it
      into existence.
    - ContentParent::AboutToLoadHttpFtpWyciwyg triggers preload on http/wyciwyg
      channel Parent::OnStartRequest via nsILocalStorageManager::Preload.
    - used by the nsILocalStorageManager exposure in LSM2.
    - Also used by the LocalStorageManager to expose the attr.

  - Used to apparently prime the CachedNextGenLocalStorageEnabled value.
  - Used in assertions:
    - localstorage::CreateQuotaClient()
    - LSObject::LSObject() constructor
    - LocalStorageManager2 constructor
    -
- CachedNextGenLocalStorageEnabled: cached any-thread version
  - Used by logic to gate:
    - QuotaManager::Init only invokes localstorage::CreateQuotaClient if
      enabled.
    - QuotaManager::EnsureStorageIsInitialized().  Changes existing call to
      MaybeRemoveLocalStorageData() to potentially call
      MaybeCreateLocalStorageArchive() if enabled.  The former wa something we
      added in preparation for downgrades of LocalStorage which will walk the
      origins, removing localstorage directories if that file existed, then
      remove the file.  (This allows us to avoid walking every time.)
    - QuotaManager client mappings pretend like the type doesn't exist if not
      enabled.
  - Assertions:
    - QM::MaybeRemoveLocalStorageData asserts not enabled.  (Documentary)
    - QM::MaybeCreateLocalStorageArchive asserts enabled.  (Documentary)




LocalStorageCommon.cpp:

LocalStorageManager2.h:
- LocalStorageManager2 implements nsIDOMStorageManager (pre-existing),
  nsILocalStorageManager (new)

LocalStorageManager2.cpp:

LSDatabase.h:

LSDatabase.cpp:

LSObject.h:

LSObject.cpp:

LSObserver.h:

LSObserver.cpp:

LSSnapshot.h:

LSSnapshot.cpp:

nsILocalStorageManager.idl:
- nsILocalStorageManager exposes:
  - nextGenLocalStorageEnabled: a pref-check via LocalStorageCommon's
    NextGenLocalStorageEnabled() caching check.
    - Used for decision-making by:
      - browser/components/extensions/parent/ext-browsingData.js to trigger
        clearing if enabled.
      - test: dom/localstorage/test/unit/test_globalLimit.js fast bails

    Note that the check is not authoritative; LocalStorageCommon also exposes
    NextGenLocalStorageEnabled() for use by C++ DOM code directly.

PBackgroundLSDatabase.ipdl:

PBackgroundLSObserver.ipdl:

PBackgroundLSRequest.ipdl:

PBackgroundLSSharedTypes.ipdlh:

PBackgroundLSSimpleRequest.ipdl:

PBackgroundLSSnapshot.ipdl:

ReportInternalError.h:

ReportInternalError.cpp:

SerializationHelpers.h:



* LocalStorageManager2
