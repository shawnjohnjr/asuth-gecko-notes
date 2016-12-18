## Overview ##

### Writes ###

Content/Child:
- IDBObjectStore.cpp's StructuredCloneWriteCallback has a JS::IsWasmModuleObject
  special-case:
- The bytecode and compiled form get serialized and stashed in BlobImplMemory
  instances.
- PBlob BlobChild actors are created for the BlobImplMemory instances via
  BlobChild::GetOrCreateFromImpl.  Because the blob impl !IsFile(),
  NormalBlobConstructorParams are sent.
  - BlobDataFromBlobImpl will retrieve the blob's internal stream, which for
    a BlobImplMemory is created via NS_NewByteInputStream in
    DataOwnerAdapter::Create.  The result is an nsStringInputStream which is
    serialized as a StringInputStreamParams in its entirety.

Parent:
- The PBlobs now manifest as BlobParent instances.  The BlobImpl will end up as
  a BlobImplStream backed by an nsStringStream.  The implementation is a little
  weird versus being directly backed by a BlobImplMemory since it will end up
  cloning the stream using assignment/copying via nsDependentCString::Assign.


Ownership transfers:
- Database::AllocPBackgroundIDBDatabaseFileParent does the
  BlobParent::GetBlobImpl and wraps the returned BlobImpl into a DatabaseFile
  that gets stored on StoredFileInfo::mFileActor.
Stream extraction:
- DatabaseFile::GetInputStream is used which invokes BlobImpl::GetInternalStream
  which is really RemoteBlobImpl::GetInternalStream which defers to its
  mBlobImpl->GetInternalStream which is BlobImplStream::GetImplStream which is
  the clone.


This should be the blob creation stack, which looks good.
- BlobParent::CreateFromParams
- CreateBlobImpl(ParentBlobConstructorParams&, BlobData&)
- CreateBlobImplFromBlobData(BlobData&, CreateBlobImplMetadata)
- CreateBlobImpl(BlobDataStream&, CreateBlobImplMetadata&)

### assert stack on blocking:

STATE OF THE ANALYSIS:
- it is in fact triggering 80470007 would block
- gdb and rr have been trouble.  more properly, rr replay is not working at all
  and the bug isn't reproducing well under just gdb.  it reproduces pretty
  easily under rr.
- code-reading is really suggesting that the above documented cases where we
  end up with a wrapped nsStringStream that can be cloned should be what we have
  and that should therefore never would_block.
  - however, if we end up with something that is not clone-able and end up with
    a pipe, that could easily end up race-prone.

NEXT STEPS:
- it seems clear that
