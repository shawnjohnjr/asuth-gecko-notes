## StorageOpenResult with null actor.
bkelly characterizes this as the error case returning a null actor.  (Which
makes sense because if we look at the success case, the actor is guaranteed
unless the child is dead, in which case there's no way the child could receive
anything.)

### Logic Path: Success ###

The Source of StorageOpenResult is Manager::StorageOpenAction, whose Complete
method invokes aListener->OnOpComplete(rv, StorageOpenResult(), CacheId).  This
inherently has a null actor, but gets fixed up...

Manager::Listener::OnOpComplete(rv, CacheOpResult, CacheId) convenience
signature is invoked which gets adapted to the full-signature virtual method
and ends up in CacheOpParent::OnOpComplete.

CacheOpParent::OnOpComplete creates an AutoParentOpResult, and invokes its
Add(CacheId, mManager).  This is what actually new()s a CacheParent and places
it in the return value.

### Logic Path: Error ###

The action will be most directly invoked in Context::ActionRunnable::Run wherein
mAction->RunOnTarget is invoked.  That'll drill down to
SyncDBAction::RunWithDBOnTarget which will invoke the actual
StorageOpenAction::RunSyncWithDBOnTarget.  The open action just returns an
nsresult, and Resolve::Resolve(rv) will be invoked by SyncDBAction (if it got
that far, otherwise the other DBAction logic would have Resolved with an error
code).

Context::ActionRunnable Resolve will get the data, dispatch itself, and then
Context::ActionRunnable::Run will invoke mAction->CompleteOnInitiatingThread.
Manager::BaseAction::CompleteOnInitiatingThread will invoke the action's
Complete method with the error-result.

In this case, that means the same thing as the success result, but mCacheId will
be INVALID_CACHE_ID.
