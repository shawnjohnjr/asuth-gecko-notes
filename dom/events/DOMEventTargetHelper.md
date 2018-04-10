DOMEventTargetHelper adds ownership/lifecycle help onto its parent class
mozilla::dom::EventTarget which handles the actual listener management and
base WebIDL exposure plus the corresponding nsWrapperCache subclassing.

## LifeCycle Magic

### Owner mechanism

DETH's register themselves with their owning globals so that the globals can
call DisconnectFromOwner on each of them when the global is going away so that
each DETH subclass can clean up after itself.  This also allows helper functions
to find existing object instances using QI/QO with predicates rather than
maintaining specific per-type bookkeeping arrays/maps (be they external, keyed
by the global, or just a big ugly soup on the nsIGlobalObject).

#### Details

mOwner tracks the most recently-associated nsIGlobalObject passed to
BindToOwner().  mOwnerWindow is also associated if the global QI's to
nsPIDOMWindowInner.  Re-binding an already bound DETH disassociates with the
previous global

nsIGlobalObject::AddEventTargetObject and
nsIGlobalObject::RemoveEventTargetObject are the exposed API calls which serve
to manipulate its mEventTargetObjects array.

The list of event targets is used for:
- nsIGlobalObject::ForEachEventTargetObject, a lambda helper.  Callers can use
  do_QueryObject and this method to find if (live) objects of a specific type
  exist and are associated with the global.  For example:
  - nsGlobalWindowInner::GetOrCreateServiceWorker
  - nsGlobalWindowInner::GetOrCreateServiceWorkerRegistration
  - nsGlobalWindowInner::AddSizeOfIncludingThis (an about:memory helper)
- nsIGlobalObject::DisconnectEventTargetObjects which uses
  ForEachEventTargetObject to invoke DisconnectFromOwner on each.  This is
  invoked by nsGlobalWindowInner when it is being destroyed/cleaned up so that
  all the now-moot objects can clean up after themselves.  It is also used by
  the base nsIGlobalObject destructor, which covers the Worker case.
  - This is the right way to clean up.  Previously, the hacky
    "inner-window-destroyed" observer notification was used in non-worker cases
    due to inconsistencies in calling DisconnectEventTargetObjects, that is
    being rectified as I type this.

DETH::DisconnectFromOwner then does all the cleanup.

### KeepAlive mechanism

DOMEventTargetHelper adds a strong refcount to itself whenever there are
listeners for any of the types KeepAliveIfHasListenersFor(type) has been called
for and has not subsequently been mooted by IgnoreKeepAliveIfHasListenersFor.
When there are no longer any relevant listeners, the strong reference is
dropped.  When DisconnectFromOwner() is called due to the global going away, the
strong reference is also dropped if it is held.

This is used in situations like where a BroadcastChannel is created, a "message"
listener is added, but the content JS code does not maintain any references to
the BroadcastChannel.  Because "message" events may be caused by other globals,
it is necessary to keep the BroadcastChannel alive as long as the "message"
listener is present, because it would be observable if we failed to keep the
BroadcastChannel alive, hence the strong reference.  However, if content JS code
added only a "foo" listener, we would not need to keep the BroadcastChannel
alive without any other content JS references because there would never be any
way for such an event to be generated.

### Details

Anytime a change happens to the set of event listeners on the DETH or the set of
types that should cause us to keep the DETH alive, MaybeUpdateKeepAlive() gets
called.  It checks if current state should keep things alive and checks against
the old state, AddRef-ing or Release-ing as appropriate.

At DisconnectFromOwner()-time or during Cycle Collection unlinking,
MaybeDontKeepAlive() is invoked which conditionally drops the refcount.