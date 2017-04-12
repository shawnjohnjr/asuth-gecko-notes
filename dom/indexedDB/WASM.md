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

### bug notes dump for re-write
(In reply to Luke Wagner [:luke] from comment #2)
> (In reply to Andrew Sutherland [:asuth] from comment #1)
>
> So I was thinking that it would be an optimization of structured clone in
> general if it was broken into two steps:
>  1. copying from the mutable JS objects into a buffer (which necessarily
> happens synchronously b/c the JS thread can immediately mutate those objects)
>  2. asynchronously writing the buffer into storage

This is how it already works with Blobs in general, with some caveats.  Because Blobs provide immutable reference-counted storage, they are referred to by reference in the structured clone stream.  However, when sending them over IPC, if the Blob doesn't already have an actor, then it may be serialized as part of the creation if the blob is small enough.  (Small Blobs' contents are sent with their actor creation.  Larger >=1MiB blobs' contents are asynchronously sent over the wire as a PSendStream.)

When the put operation is performed on the database's I/O thread, a stream is requested from the Blob so that the put can complete asynchronously, which can include waiting for the contents of the blob to be streamed up from the child.  That's how we had the WOULD_BLOCK error; the child had fallen behind on streaming the contents of the blob to the parent.

The likely performance slowdown for WASM right now is that the Blob doesn't already exist.  At http://searchfox.org/mozilla-central/source/dom/indexedDB/IDBObjectStore.cpp#361 the "serialize" call I assume is new()ing 2 giant buffers and memcpy()ing the contents from the JS backing store.  Things would be faster if the JS code could expose a reference-counted memory buffer that a shared MemoryBlobImpl could reference instead of needing to duplicate the memory.

> An alternative impl strategy that would actually be quite symmetric with
> alternative data streams would be if, when a JS::WasmModule was serialized,
> IDB immediately wrote the bytecode but, for the machine code file, IDB
> simply handed the JS::WasmModule an nsIOutputStream that could be written to
> at some point in the future.  The main challenge I suppose is handling
> concurrent reads to the nsIOutputStream write; the ideal thing here would be
> for reads to wait on the nsIOutputStream to close since they'll otherwise
> have to recompile the machine code anyway.

There is a StreamBlobImpl that you could hand the nsIInputStream end of a pipe to to be its contents.  The problem is that since the Blob is defined to be immutable, we need to know its size when the blob is created.  This is because the Blob's size gets encoded into the structured clone data stream and is the authoritative source of this information.  I'm assuming the background compilation doesn't know the resulting size before it finishes?

> > And instead we
> > can concentrate our engineering efforts on making the DOM Cache API support
> > alternative data streams, which seems like the preferable long-term solution
> > for many reasons.
>
> The important resolution of that discussion, though, was that while implicit
> caching will be great for handling the many-small-wasm-libraries case, it
> will break down for (1) really big modules which are too big for the cache
> (who of course are when we need caching the most; we're talking 100-200mb
> cache files here) (2) caching modules whose bytes are computed and thus have
> no backing cache entry (Unity actually already does this so it can apply
> custom compression, so this is already a critical use case).  For these
> cases, IDB is the power tool that saves the day.

I think there's some confusion here.  I'm referring to the DOM Cache API (https://developer.mozilla.org/en-US/docs/Web/API/Cache) that's currently part of the ServiceWorker spec, not the HTTP Cache.  It's quota-manager backed storage with the same storage limits as the IndexedDB API.  Although its storage keys are fetch Request instances (which support crazy HTTP "Vary" header matching rules) and so need to look like valid http(s) URL's, they don't have to actually correspond to anything on the web.  You can do cache.put("http://example.com/foo", new Response(new Blob([someArrayBuffer]))) that doesn't exist on the web and no one will be the wiser.

The appealing things about DOM Cache for this use-case are:
- It doesn't have a transaction model complicating things.  (There is a proposal for transactions, but it amounts to a mutex/lock primitive for coordinating access to the cache for interested logic.)
- 
