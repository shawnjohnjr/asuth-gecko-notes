There's a lot going on with Blobs.

These notes have not been updated for the file splittings that occurred, nor
for the PBlob overhaul in https://bugzilla.mozilla.org/show_bug.cgi?id=1353629
which moves from managed actors-per-blob to more inert IPDL structs with
(either direction) send stream actors handling handling the more complicated
cases.

## Roll Call ##

### Local (non-IPC) ###

dom/file/nsIDOMBlob.idl:
* nsIDOMBlob (isa nsISupports): empty interface

dom/file/File.h:
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

dom/file/MultipartBlobImpl.h:
* MultipartBlobImpl (isa BlobImplBase)

dom/file/MutableBlobStorage.h: (evolved from BlobSet)
* MutableBlobStorage (not isa): explicitly main-thread-only.  Helper class that
  creates and writes to a temporary file on an I/O thread, producing a
  BlobImplTemporaryBlob asynchronously on request.  (A runnable bounce is used
  to ensure all the data has hit disk before creating the blob and invoking the
  callback.)

dom/indexeddb/ActorsParent.cpp:
* BlobImplStoredFile (isa BlobImplFile)

### IPC-related ###

dom/file/ipc/PBlob.ipdl:
* PBlob: Actor for blobs:
  * ResolveMystery (c->p):

dom/file/ipc/PBlobStream.ipdl:
* PBlobStream: Async means of child asking the parent for {InputStreamParams,
  OptionalFileDescriptorSet} tuple; compare with BlobStreamSync provided by
  PBlob which does the same thing.

dom/file/ipc/nsIRemoteBlob.h (not .idl!):
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
* MysteryBlobConstructorParams (p->c): empty.  Used only for IndexedDB blobs
  where the child is expected to PBlob.ResolveMystery() once it deserializes the
  missing size/date metadata from the structured clone.
* KnownBlobConstructorParams (p<-c): struct { nsID }.  Used to avoid refcounting
  races when a BlobChild is created off of an existing (remote) blob (identified
  by its nsID).  We addref the "other" BlobImpl and only free it once the parent
  has processed our constructor and sent us a PBlob.CreatedFromKnownBlob()
  message.  **understand/characterize race; this is currently speculative:**.
  My current best speculation is that this becomes an issue with a remote Blob
  that is surfaced on both the main thread and a worker (and possibly PContent
  versus PBackground) and therefore they're using different channels and the
  ordering of [add, release] would not otherwise be guaranteed.
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

### General Flow ###

When sending a blob over the wire, it's sent as a PBlob using the
BlobConstructorParams from DOMTypes.ipdlh.  The rule is that the parent always
"knows" the content of all blobs, but the child has to ask for the contents
(via PBlobStream or BlobStreamSync)

### Parent Blobs Sent From Child ###

These come through BlobParent::CreateFromParams which uses the (parent-only)
CreateBlobImpl methods from ipc/Blob.cpp.

For memory-backed blobs authored in the content process, these end up as
BlobImplStream instances backed by either an nsStringStream for small blobs or
an unbounded nsPipeInputStream filled-by a (P)SendStream(Parent).  This is
notably different than a BlobImplMemory because for:
- nsStringStream, cloning uses assignment/copying via
  nsDependentCString::Assign.
- nsPipe(InputStream): It's clever about clones.  Note that because the pipe is
  explicitly infinite, we don't have to worry about backpressure killing the
  send stream.


#### Blobs sent from the child to the parent multiple times ####

Unless this is IndexedDB we're talking about, Blobs sent from a child to a
parent that aren't remote blobs sent from the parent will be duplicated every
time they are sent up to the parent.

IndexedDB use a weak-ref cache that maps Blob instances to previously created
(and still alive) actors in IDBDatabase::GetOrCreateFileActorForBlob.

When the parent receives a blob it creates a UUID that it tags the Blob with
(in BlobParent::CreateFromParams).  This allows the

### Mystery Blobs and IndexedDB ###

Mystery blobs are created by the parent in BlobParent::GetOrCreateFromImpl when
the size or date of the blob is unknown.  This is expected to occur in the case
of IDB's BlobImplStoredFile.  It is left up to the child to send the information
up when it deserializes the Blob in CommonStructuredCloneReadCallback via calls
to CreateAndWrapBlobOrFile().  The structured clone representation serializes
the size (and date, if file) as part of the structured clone, which helps avoid
the need to perform any fstat's.  (Not to mention it might be impossible to
convince the filesystem to use the date we want it to.  Not that the data
couldn't have just been stored in the database outside of the structured clone
in a normalized table like where IDB already stores the refcounts.)

### Specific Blob Type Handling ###

#### Memory-backed Blobs ####

Memory-backed Blobs (BlobImplMemory) are exposed as nsStringInputStreams via
DataOwnerAdapter::Create using NS_NewByteInputStream.
nsStringInputStream::Serialize serializes as StringInputStreamParams where the
entirety of the blob is sent across the wire in a single go.

### Implementation Details / Further Roll Call ###

#### Blob.cpp ####

See dom/ipc/Blob.md
