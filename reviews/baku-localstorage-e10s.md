## Multiprocess localstorage change event broadcasting ##
https://bugzilla.mozilla.org/show_bug.cgi?id=1285898

### IPC

PContent gains a bi-directional DispatchLocalStorageChange(nsStrings:
documentURI, key, oldValue, newValue; Principal) signature with no validation
checks.

#### (Initial) Broadcast ####

For changes initiated in the child, in DOMStorage::BroadcastChangeNotification:
* If in a child process, it's local (not session) storage, and there's a
  principal (invariant guard; should only occur if there wasn't an mWindow on
  the DOMStorage)...
  * Invoke SendBroadcastLocalStorageChange (which will not bounce back down to
    us.)
* Always invoke the prior code from BroadcastChangeNotification that has now
  been refactored out into the static DOMStorage::DispatchStorageEvent.

For changes initiated in the parent, in DOMStorage::DispatchStorageEvent:
* If the storage type is local storage and in the parent process and there's a
  principal (again, invariant guard)...
* Send to all children using AllProcesses(eLive) just like in ContentParent's
  RecvDispatchLocalStorageChange.

**WEIRD**
This results in the somewhat weird asymmetry that if we have iframes "p", "c1",
and "c2" in the parent and two separate processes, all for the same origin, that
"p" will not see events from "c1" and "c2" but they will see events from "p".


#### Propagation ####

The ContentParent implementation uses ContentParent::AllProcesses(eLive) to
re-broadcast the event to all children but the originating child.  The
enumeration crosses process-types ("file", "web", large allocation, etc.) and
has no guards anywhere.

The ContentChild implementation invokes the new static
DOMStorage::DispatchStorageEvent with a hardcoded type of
DOMStorage::LocalStorage and a null DOMStorage.

### Event Dispatching

The static DOMStorage::DispatchStorageEvent has been extracted out from
the member function BroadcastChangeNotification.

There is no change in the observer signature other than the change to
StorageEvent.


StorageEvent is altered to optionally hold a strong nsIPrincipal mPrincipal.

nsGlobalWindow's processing is altered to:
* pull the storagePincipal out of the event instead of out of the event's
  "StorageArea" DOMStorage.
* differentiate between local/session based on aData (which was always either
  u"sessionStorage"/u"localStorage" intead of on the DOMStorage::GetType() val.

### Change Propagation

ApplyEvent makes the change visible.  As of the initial review draft, it does
this by using a new explicit handler to invoke mutations on the DOMStorageCache
in a way that wastefully/dangerously causes the writes to be propagated back up
to the database.

ApplyEvent is hacked into CloneStorageEvent which sorta has the upside that it
occurs prior to the IsFrozen() check so that the mutations will occur at the
right time.  Unfortunately, because the event is sent through an additional time
on unfreeze
