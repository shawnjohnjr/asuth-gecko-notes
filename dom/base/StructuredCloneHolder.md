## JS Structured Clone Overview ##

The JS engine's StructuredClone.h defines the struct JSStructuredCloneCallbacks
which define read, write, reportError, readTransfer, writeTransfer, and
freeTransfer ops.

### Memory Handling ###

JSStructuredCloneData isa mfbt mozilla::BufferList

## StructuredCloneHolderBase ##

Base adapter layer.

The JSStructuredCloneCallbacks are adapted to the StructuredCloneHolder via
static helpers that cast the void* aClosure into a StructureCloneHolderBase*
and invoke CustomReadHandler, CustomWriteHandler, CustomReadTransferHandler,
CustomWriteTransferHandler, CustomFreeTransferHandler.

High level Read() and Write() methods are exposed that add assertions and
otherwise just adapt calls to pass the static JSStructuredCloneCallbacks.

### Memory Handling ###

Holds a protected UniquePtr<JSAutoStructuredCloneBuffer> mBuffer whose data() is
exposed via BufferData(), where data() is a JSStructuredCloneData instance.
By not directly exposing the JSAutoStructuredCloneBuffer, access to adopt() /
steal() / abandon() is prevented, which is significant since only those are
friends to meddle with JSStructuredCloneData's ownTransferables_.

## StructuredCloneHolder ##

Introduces support for cloning and transferables.

### Memory Handling ###

Just uses StructuredCloneHolderBase's mBuffer.

Subclasses can change this up by re-implementing Read and Write and basing them
on top of ReadFromBuffer/Write-then-steal hacks, like StructuredCloneData does.
(Remember that mBuffer is protected so they have access to adopt/steal/etc.)
