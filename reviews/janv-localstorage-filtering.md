https://bugzilla.mozilla.org/show_bug.cgi?id=1462162
filtering implementation

## Overview

LocalStorageCache gets its own protocol, PBackgroundLocalStorageCache for
propagating changes.  These actors are tied to specific origins and are
exclusively for (filtered) change broadcasting.  All communication with the
database continue to be orthogonal and occur over PBackgroundStorage.

Correctness in terms of ordering and races remains un-changed.  All that's
happening is that the one giant broadcast pipe has been split up into many
smaller per-origin pipes.  Security guarantees also remain unchanged.

## Details

### Coordination / Lifecycles

In content (be it in a content process or the parent process for non-e10s or
chrome-privileged documents), LocalStorageManager::GetStorageInternal creates a
non-refcounted LocalStorageCacheChild whenever it creates a LocalStorageCache.
The LocalStorageCache is expected to have its refcount hit 0 when there are no
longer any interested windows holding LocalStorage instances.  The
LocalStorageCache destructor invokes
LocalStorageCacheChild::SendDeleteMeInternal which does the shutdown roundtrip
ping-pong and synchronously drops both pointer edges via invoking
mCache->ClearActor() and clearing mCache.

In the parent, LocalStorageCacheParent instances are created corresponding to
the LocalStorageCacheChild instances in the content processes.  They are
tracked by gLocalStorageCacheParents which is keyed by the origin key already
used by LocalStorageManager::mCaches.  The values differ in that it stores an
array of LocalStorageCacheParent* instances.  Note that while LSCP instances are
refcounted, no refcounts are ever taken by anything other than IPC code.  The
array membership is safe because LSCP::ActorDestroy removes the entry.

Note that leakage over time is avoided by:
- removing arrays once they're empty
- nulling out the map if it ever becomes empty.  (This is good for hashtables
  that don't automatically shrink themselves, however, nsTHashtable does...)

### General Propagation Flow

- LocalStorage mutation happens.
  - This calls the LocalStorageCache method.
    - These calls invoke NotifyObservers directly (as long as this wasn't
      applying a propagated change).
      - NotifyObservers sends a "Notify" message to the parent.
  - As long as the cache call did not result in a no-op, we invoke OnChange.
    (Previously this was BroadcastChangeNotification, and it would also send the
    change up to the parent.  This is now handled above us in NotifyObservers.)
    - OnChange invokes NotifyChange.
      - NotifyChange is the same as it was before.  It creates a StorageEvent
        and:
        - Uses StorageNotifierService::Broadcast to send it to all involved
          windows.
        - Creates the StorageNotifierRunnable that's used by devtools.
- The parent receives the message from LocalStorageCacheChild sent by
  NotifyObservers.
  - It iterates over all the listening managers that aren't itself and invokes
    sends an "Observe" message to the other children.
- Some other child receives a RecvObserve message.
  - RecvObserve invokes Storage::NotifyChange.  Again, this is the same as
    before (but previously the call happened via
    BackgroundChildImpl::RecvDispatchLocalStorageChange invoking
    LocalStorage::DispatchStorageEvent which has been refactored out of
    existence to instead NotifyChange directly.  DispatchStorageEvent was
    basically legacy indirection with no purpose.)

## Patch Minutae

### IPC
PBackground IPC changes:
- to-parent PBackgroundLocalStorageCache added; constructor takes
  (principalInfo, originKey, privateBrowsingId)
- to-parent BroadcastLocalStorageChange removed.
- to-child DispatchLocalStorageChange with same signature as above removed.

PBackgroundLocalStorageCache added:
- PBackground-managed (consistent with above)
- to-parent Notify(documentURI, key, oldValue, newValue) was the first 4 args
  of BroadcastLocalStorageChange.  The other 2 were (principalInfo, isPrivate)
  which are covered by the constructor (and more; it gets originKey plus the
  specific private browsing id instead of isPrivate).
- to-parent DeleteMe() is standard IPC round-trip termination to avoid the
  parent sending unsolicited messages to the child which cross a deletion.
- to-child Observe() added.  Unlike Notify, this has the excess info of
  principalInfo and privateBrowsingId. **why**

### Core impl minutae

LocalStorage.h:
- DocumentURI() infallible accessor added.
- static DispatchStorageEvent removed.
- s/BroadcastChangeNotification/OnChange/ on the method.

LocalStorage.cpp:
- s/BroadcastChangeNotification/OnChange/ at call-sites.  For the method itself,
  we now call NotifyChange directly, skipping the now-removed static
  DispatchStorageEvent (which was also used by
  BackgroundChildImpl::RecvDispatchLocalStorageChange).  The logic to invoke
  SendBroadcastLocalStorageChange if we could extract a principal has also been
  removed. **to where**

LocalStorageCache.h:
- AssertIsOnOwningThread() using NS_ASSERT_OWNINGTHREAD mechanism.
- SetActor/ClearActor/mActor for LocalStorageCacheChild.
- NotifyObservers(LocalStorage*, key,old,new) added, which directly invokes
  SendNotify on the actor (which is guarded).
- Calls to NotifyObservers added to the mutation operations.  Control flow
  slightly changed to early return after applying the change to the local cache
  if we're propagation instead of an actual content change (which therefore
  needs to be propagated.)

LocalStorageCache.cpp:
- Actor lifecycle:
  - nullptr in constructor
  - SetActor() sets with thread-check and rising-edge assertion-checks
  - destructor does SendDeleteMeInternal if there's still an actor, with
    assertion check that the weak reference gets cleared.

StorageIPC.h:
- LocalStorageCacheChild implemented, NOT ref-counted, yes owning-threaded.
- LocalStorageCacheParent implemented, YES inline-refcounted.


BackgroundParentImpl.{h,cpp}:
- Alloc/Constructor/Dealloc added for PBackgroundLocalStorageCacheParent.  Hands
  off to methods in StorageIPC.{h,cpp}.
- Removed RecvBroadcastLocalStorageChange.  The new equivalent is
  LocalStorageCacheParent::RecvNotify which is where the heart of the filtering
  logic happens thanks to gLocalStorageCacheParents performing lookup based on
  origin key.

BackgroundChildImpl.{h,cpp}:
- Alloc/Dealloc added for PBackgroundLocalStorageCacheChild, which is not
  refcounted.
- Removed RecvDispatchLocalStorageChange which invoked the static
  LocalStorage::DispatchStorageEvent.  The new equivalent is in
  LocalStorageCacheChild::RecvObserve.
