NB: There's a separate MessageChannel implementation in dom/ipc/ that is part of
the mozilla::ipc namespace, contrast with mozilla::dom for this one.

## Big Picture ##

Managed via PBackground.

If invoked on a worker, creates a MessagePortWorkerHolder to keep the worker
alive since it's now able to hear things.  If invoked on the main thread/a doc,
registers on inner-window-destroyed.

MessagePorts/MessagePortChild:
* Only involve PBackground when shipped/entangling.  If a MessageChannel is
  created by JS code, the state is eStateUnshippedEntangled which is
  same-thread and just directly reaches through to the other port (which it
  learns about via UnshippedEntangle()).

MessagePortParent/MessagePortService:
* MessagePortService is the brains.  It maintains the map, has the actual logic
  for entangling, disentangling (including sequence bumping), and message
  transformation for the core part of the actual postmessage-ing.
* MessagePortParent adds little above the fundamental Actor lifecycle.


Characterized by source and destination UUID's.

## Participants ##

Alphabetical:
* MessageChannel: WebIDL spec'ed; simple binding to expose both ports.  Creates
  2 unshipped entangled ports that directly communicate without involving
  PBackground
...

## Structured Cloning Transfer of Ports ##

Writing:
* StructuredCloneHolder::CustomWriteTransferHandler explicitly checks for
  MessagePort instances and invokes CloneAndDisentangle.

Reading:
* StructuredCloneHolder::CustomReadTransferHandler uses the explicit tag to
  MessagePort::Create() using the MessagePortIdentifier signature which implies
  entangling.

(Freeing):
* StructuredCloneHolder::CustomFreeTransferHandler does MessagePort::ForceClose.

## PostMessage ##

### Send ###

Simplified flow:
* Creates SharedMessagePortMessage.
* Bails if appropriate to bail (after transferring for consistent side effects?)
* If eStateUnshippedEntangled meaning other end is same-thread, reach in and
  directly append and dispatch.
* Queue in mMessagesForTheOtherPort if eStateEntangling
* Otherwise, do mActor->SendPostMessages() having used
  SharedMessagePortMessage::FromSharedToMessagesChild on an AutoTArray we just
  created.

### Receive ###

PostMessageRunnable::DispatchMessage isa CancelableRunnable does the work.

Gets the globalObject off

## Lifecycle ##

MessagePortWorkerHolder
