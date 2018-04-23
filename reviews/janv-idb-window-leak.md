## Bug 1410420 - support.microsoft.com ghost window via IDBDatabase object on 57 beta
https://bugzilla.mozilla.org/show_bug.cgi?id=1410420

### WIP fix feedback notes

Minutae:
- BackgroundFactoryRequestChild gains weak
  BackgroundDatabaseChild* mDatabaseActor. **NEEDS ANNOTATION**
  - Initialized to null at construction.
  - Set by new SetDatabaseActor method.  Method has clever rising/falling-edge
    assertion so it can't go from valid pointer to valid pointer.
    - Invoked by BackgroundDatabaseChild::EnsureDOMObject() when it creates the
      mTemporaryStrongDatabase and saves to mDatabase as well, setting the actor
      to BackgroundDatabaseChild.
  - When valid, used to invoke mDatabaseActor->ReleaseDOMObject() in
    BackgroundFactoryRequestChild::HandleResponse(nsresult), the error-handling
    overload.
    - HandleResponse
