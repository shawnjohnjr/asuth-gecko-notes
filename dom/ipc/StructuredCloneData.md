IPC-aware StructureCloneHolder subclass that, in addition to providing IPC
serialization support (by just deferring to the underlying
JSStructuredCloneData), adds additional memory-management glue.

### Memory Allocation ###

StructuredCloneHolder itself just holds a UniquePtr-owned JSStructuredCloneData.

StructuredCloneData can get its data from either mExternalData, a somewhat
mis-named JSStructuredCloneData instance, or in RefPtr<SharedJSAllocatedData>
mSharedData, where SharedJSAllocatedData is defined in the same header.

This gives us two construction-based modes of operation consuming existing data:
- `UseExternalData`: Given an existing JSStructuredCloneData whose life-cycle
  will outlast ours, use BufferList::Borrow to create a non-owning
  read-only-view BufferList which we populate mExternalData with.  To reiterate,
  there is no reference counting with this option; if the external data gets
  freed, everything explodes.  There is no transfer of ownership!  Used by:
  - nsFrameMessageManager.cpp's UnpackClonedMessageData whose callers then go
    onto subsequently call a ReceiveMessage variant that does the actual Read.
  - BroadcastChannelChild which immediately after calls Read.
- `CopyExternalData`: Uses SharedJSAllocatedData::CreateFromExternalData to
  copy the contents of the provided raw (const char* aData, size_t aDataLength)
  buffer.  Seems very targeted at the one existing consumer:
  - nsStructuredClone::InitFromBase64 which writes its data into an
    nsAutoCString which won't survive the return from the function call.  (And
    the use-case there is script-accessible structured cloning, so there is no
    call where the data could be synchronously used.)

And one IPC read mode of operation:
- `ReadIPCParams`: A SharedJSAllocatedData instance is created using Move() of
  the JSStructuredCloneData that ReadParam was used to directly deserialize
  into.

And one Write mode of operation (it of course starts out empty if you're writing
into it):
- `Write`: StructuredCloneHolder::Write is used and then its mBuffer is stolen
  and a SharedJSAllocated is initialized with move semantics.
  **bugcheck** abandon() is used followed by steal() without using adopt(),
  which means that ownership of transferables is potentially in question.  I'm
  assuming the answer is that this isn't currently a problem since we're
  operating as different-process and either the existing callers don't pass
  transferables or something else higher level handles it.  (Or there just
  aren't tests...)
