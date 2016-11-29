Managed via PBackground.

Similar to MessageChannel/MessagePort implementation but simpler and slightly
more modern in terms of types used.

* BroadcastChannel has the client brains.  Unlike MessagePort it always binds
  to PBackground because it inherently must.
* BroadcastChannelChild is the minimal child actor help by the BC.
* BroadcastChannelParent is the minimal parent actor that talks to the
  BroadcastChannelService.
* BroadcastChannelService is the routing/registration brains.

## Dispatch ##

BroadcastChannelChild::RecvNotify is the interesting bit.  Differentiates
between main thread and worker dispatch based on what thread it's on.  If not on
main it uses GetCurrentThreadWorkerPrivate(), code unifies then once it has a
global.  Compare with the MessagePort case which uses DOMEventTargetHelper as
well but initializes it using the nsIGlobalObject constructor so it can pull it
out against.  BroadcastChannel uses the nsPIDomWindowInner constructor and so
the global just ends up null, requiring this footwork.

Provides origin to event from its own origin; this is per the HTML spec (and an
invariant verified by PBackground initializer and safe).

## IPDL ##

### Payload Differences v MessagePort ###

Summary: It looks like MessagePort is overdue for a normalization and cleanup to
use ClonedMessageData.

MesagePort has its own type:
    struct MessagePortMessage
    {
      MessagePortIdentifier[] transferredPorts;
      uint8_t[] data;
      PBlob[] blobs;
    };

BroadcastChannel uses the common ClonedMessageData from DOMTypes.ipdlh:
    struct ClonedMessageData
    {
      SerializedStructuredCloneBuffer data;
      PBlob[] blobs;
      MessagePortIdentifier[] identfiers;
    };

Where SerializedStructuredCloneBuffer is a native class of `uint64_t* data`, and
`size_t dataLength` with a template that knows how to write its length followed
by its data padded out to 64-bit alignment.

## Flow ##

### Sending ###

* BroadcastChannel::PostMessage: does a state check, then defers to:
* BroadcastChannel::PostMessageInternal: Creates a BroadcastChannelMessage (isa
  StructuredCloneHolder that disables transferring, which differs from others
  like MessagePort), structured clone writes into it, then invokes:
* PostMessageData, which takes the BroadcastChannelMessage and ensures that a
  BCPostMessageRunnable is eventually created and dispatched to the current
  thread.  This happens whenever we have mActor; if we have it right now, then
  now, otherwise the message is appended to mPendingMessages will queue it
  until ::ActorCreated() happens.
* BCPostMessageRunnable::Run:
  * sanity-checks mActor isn't destroyed, bailing if it is.
  * Creates a stack-owner ClonedMessageData, borrowing (which means true owner
    has to stay alive, which it does because this all happens in a single call)
    from the BroadcastChannelMessage.
  * Sizes ClonedMessageData's blobsChild via SetCapacity(), then runs over the
