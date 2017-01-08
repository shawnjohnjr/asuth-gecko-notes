## Multiprocess localstorage change event broadcasting ##
https://bugzilla.mozilla.org/show_bug.cgi?id=1285898

### IPC Propagation

PContent gains a bi-directional DispatchLocalStorageChange(nsStrings:
documentURI, key, oldValue, newValue; Principal) signature with no validation
checks.

### Event Dispatching

There is no change in the observer signature other than the change to
StorageEvent.


StorageEvent is altered to optionally hold a strong nsIPrincipal mPrincipal.

nsGlobalWindow's processing is altered to:
* pull the storagePincipal out of the event instead of out of the event's
  "StorageArea" DOMStorage.
* differentiate between local/session based on aData (which was always either
  u"sessionStorage"/u"localStorage" intead of on the DOMStorage::GetType() val.
