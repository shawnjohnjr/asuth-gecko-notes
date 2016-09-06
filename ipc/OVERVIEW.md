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
