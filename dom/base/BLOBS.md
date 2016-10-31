There's a lot going on with Blobs.

## Roll Call ##

### Local (non-IPC) ###

dom/base/nsIDOMBlob.idl:
* nsIDOMBlob (isa nsISupports): empty interface

dom/base/File.h:
* Blob (isa nsIDOMBlob)
* File (isa Blob)
* BlobImpl (isa nsISupports): abstract class
* BlobImplBase (isa BlobImpl)
* BlobImplMemory (isa BlobImplBase): explicitly defined to be usable off the
  main thread.  Actual memory storage handled by DataOwner (isa
  LinkedListElement) which allows shared memory use.
* BlobImplTemporaryBlob (isa BlobImplBase): presumably what it says on the can?
* BlobImplFile (isa BlobImplBase): nsIFile under the hood.
* EmptyBlobImpl (isa BlobImplBase)
* BlobImplStream (isa BlobImplBase): nsIInputStream under the hood.

dom/base/MultipartBlobImpl.h:
* MultipartBlobImpl (isa BlobImplBase)

dom/base/MutableBlobStorage.h: (evolved from BlobSet)
* MutableBlobStorage (not isa): explicitly main-thread-only.  Helper class that
  creates and writes to a temporary file on an I/O thread, producing a
  BlobImplTemporaryBlob asynchronously on request.  (A runnable bounce is used
  to ensure all the data has hit disk before creating the blob and invoking the
  callback.)

dom/indexeddb/ActorsParent.cpp:
* BlobImplStoredFile (isa BlobImplFile)

### IPC-related ###

dom/ipc/nsIRemoteBlob.h (not .idl!):
* nsIRemoteBlob (isa nsISupports NS_NO_VTABLE): provides GetBlobChild() and
  GetBlobParent().

dom/ipc/DOMTypes.ipdlh:
* BlobDataStream: struct { IPCStream stream, uint64_t length }
* BlobData: union of (nsID, BlobDataStream, BlobData[])
* OptionalBlobData: union of (BlobData, void_t)
constructor params:
* NormalBlobConstructorParams (p<>c): struct {contentType, length,
  OptionalBlobData}.  When p->c, OptionalBlobData is void_t, when c->p it's
  BlobData.
* FileBlobConstructorParams (p<>c): struct {name, contentType, path, length,
  modDate, isDirectory, OptionalBlobData}.  Same deal as
  NormalBlobConstructorParams, when p->c, optionalBlobData is void_t, when c->p
  it's BlobData.
* SlicedBlobConstructorParams (p<-c): struct {PBlob source, nsID, begin, end,
  contentType}.
* MysteryBlobConstructorParams (p->c): empty.
* KnownBlobConstructorParams (p<-c): struct { nsID }
* SameProcessBlobConstructorParams (p<>c): struct {intptr_t addRefedBlobImpl}
  where pointer gets reinterpret_cast<BlobImpl>-ed.
* AnyBlobConstructorParams: union of all of the above
* ChildBlobConstructorParams: struct {nsID, AnyBlobConstructorParams blobParams}
* ParentBlobConstructorParams: struct {AnyBlobConstructorParams blobParams}.
* BlobConstructorParams: union (ChildBlobConstructorParams,
  ParentBlobConstructorParams)

dom/ipc/BlobParent.h:
* BlobParent (isa PBlobParent)

dom/ipc/BlobChild.h:
* BlobChild (isa PBlobChild)

dom/ipc/Blob.cpp:
* BlobInputStreamTether (isa serializable seekable multiplex inputstream):
  "keep a blob alive at least as long as its internal stream"
* RemoteInputStream (isa seekable serializable inputstream):
  Capture current thread at creation time as mEventTarget.  Uses a monitor to
  **finish summarizing**
  2 modes:
  * ???: Takes a BlobImpl, no BlobChild actor.
  * WorkerStream: Takes BlobImpl and BlobChild actor:
* InputStreamChild (isa PBlobStreamChild): holds RemoteInputStream
* InputStreamParent (isa PBlobStreamParent): holds fd's

also check: StructuredCloneHolder.cpp's usage of MultipartBlobImpl::Create

## IPC ##

### Blob.cpp ###

See dom/ipc/Blob.md
