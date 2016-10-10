## Threads: Who, What Where ##

## Actor Destruction ##

Children get ActorDestroy()ed before their parents.  Makes sense.  (But paranoia
demands I state this for myself.)

In PContentParent::OnChannelClose, we have:
    DestroySubtree(NormalShutdown);
    DeallocSubtree();
    DeallocShmems();
    DeallocPContentParent();

With PContentParent::DestroySubtree explicitly destroying kids then itself:
    kids->DestroySubtree(subtreewhy);
    ActorDestroy(why)

## Idioms ##

### Allocation versus constructor ###

Covered by
https://developer.mozilla.org/en-US/docs/Mozilla/IPDL/Best_Practices#When_to_run_code

The short story is stick to allocation for Alloc to the extent possible and do
the actual work in the constructor.

In a situation where it's a "child: PProtocol" scenario, I've seen bkelly skip
the explicit allocation step and have:
* The Alloc boilerplate just return null.
* The causal method explicitly newing the BlahParent and then invoking
  SendPBlahConstructor.

This all makes sense.

### Return Values ###

### When to Run Code ###

https://developer.mozilla.org/en-US/docs/Mozilla/IPDL/Best_Practices#When_to_run_code

* Manager::AllocPProtocol - allocation
* Manager::RecvPProtocolConstructor - initialization, protocol deletion (the TypeAheadFind protocol uses one-shot protocols like this)
* Actor::Recv__delete__ - cleanup, IPDL calls still valid but ill-advised
* Actor::ActorDestroy - non-IPDL cleanup
* Manager::DeallocPProtocol - deallocation
