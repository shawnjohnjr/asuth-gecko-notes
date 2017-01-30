
## Create Blob implementations from BlobData serialization type ##

Note: All of these methods assert that they are in the parent process!

CreateBlobImpl(nsID, CreateBlobImplMetadata):
* Asks BlobParent::GetBlobImplForID to provide a BlobImpl, which it returns.
* Asserts the blob is not mutable.

CreateBlobImpl(BlobDataStream, CreateBlobImplMetadata):
* Uses DeserializeIPCStream()
* Asks stream for length, then:
  * If it's a file AKA has-a-name (and we're using it because we aren't a nested
    call from inside the BlobData[] variant):
    * If it has a length, uses BlobImplStream
    * No length, uses EmptyBlobImpl
  * Not a file, but has length, uses BlobImplStream
  * Not a file, no length, uses EmptyBlobImpl

CreateBlobImpl(BlobData[], CreateBlobImplMetadata):
* Special-case length of 1 directly calls CreateBlobImplFromBlobData and returns
  it directly.
* Seems to use CreateBlobImplMetadata.mHasRecursed to ensure that only the
  outermost blob gets is treated as a File (in terms of having a name).
* Loops over the BlobData instances, using CreateBlobImplFromBlobData for each,
  then passing to MultipartBlobImpl::Create (File/non-File as appropriate).

CreateBlobImplFromBlobData(BlobData, CreateBlobImplMetadata):
* Switches on BlobData.type() to dispatch to either of the 3 above
  CreateBlobImpl calls.

## Create BlobData serialization from BlobImpl's ##

BlobDataFromBlobImpl(templated ChildManagerType*, BlobImpl*, BlobData, AutoIPCStream[]):
* If BlobImpl->GetSubBlobImpls(), creates BlobData[] and iterates over the
  sub-impls, invoking BlobDataFromBlobImpl recursively on each.
* QueryInterfaces BlobImpl to nsIRemoteBlob, if successful, invokes
  GetBlobChild() and returns its nsID ParentID() (which is one of the union
  types).
* Falling through, BlobImpl->GetInternalStream is used to construct/serialize an
  AutoIPCStream which is given (via TakeValue()) to a BlobDataStream (which is
  a struct of the returned IPCStream and the length which was extracted via
  BlobImpl->GetSize()).  (Which, reminder, is a bundle of InputStreamParams and
  any related fd's OR a PSendStream which is a thing for this child->parent
  case.)


### RemoteInputStream ###
RemoteInputStream:
* Constructor modes:
  * WorkerStream: When it has an actor.
  * uh, other
* ~RemoteInputStream:
  * If we're not on the owning thread, explicitly null out mStream and
    mWeakSeekableStream.  And if (mBlobImpl), release on the owning thread.
    (If Close was called, mBlobImpl is now null.)
  * If we were on the owning thread, it seems like the default destructors will
    handle cleaning up mStream and mBlobImpl as relevant.  (mWeakSeekableStream
    was a non-owning QI'd mStream dupe pointer and so needs no cleanup.)
* SetStream(nsIInputStream):
  * Assert on the owning thread.
  * QI the nsIInputStream to an nsISeekableStream.  Assert that this didn't
    somehow produce a different instance (custom QI dubiously using GetInterface
    semantics instead of QI semantics?) if it was seekable.
  * With the lock held, if the stream isn't already set, set it.  Also, assert
    that if we already had a stream, that we're a WorkerStream (we had an Actor)
    provided.  **determine why that could ever happen**
* BlockAndWaitForStream():
  * If on the owning thread, do the actual work:
    * Assert we have an actor via IsWorkerStream.
    * NS_WARNING if we're the main thread, since it's not okay to issue sync
      calls from the main thread.
    * Synchronously invoke SendBlobStreamSync which returns an InputStreamParams
      and an OptionalFileDescriptorSet and convert it into an nsIInputStream via
      DeserializeInputStream.
    * Invoke SetStream with that stream, return.
  * Since not on the owning thread, wait for the owning thread to do the work
    by calling ReallyBlockAndWaitForStream().
* ReallyBlockAndWaitForStream():
  * Assert not on the owning thread.
  * Acquire the monitor and keep spinning (notified by SetStream after setting
    mStream) until we have mStream.  (Assert the stream on exit, and in debug
    assert the seek position is 0.)
  * Return (control flow)
* IsSeekableStream():
  * If on the owning thread, return true if we don't have the stream yet,
    otherwise return whether we have a (weak) seekable interface for it.
  * If not on the owning thread, use ReallyBlockAndWaitForStream().
* Close():
  * Use BlockAndWaitForStream to make sure the stream was ever spun up (works
    on either thread).
  * Swap mBlobImpl to a stack-held refptr to outlive the call to close, I guess.
  * Invoke mStream->Close().
* Available(out uint64_t*):
  * If not on the owning thread, block, then return mStream->Available().
    **BUG: code clearly wants to early return, but does not, and falls through**
  * Since not on the owning thread, grab the lock in order to copy off mStream.
  * If we had it, invoke Available() and return immediately.
  * Didn't have it, check if Close() has been invoked (mBlobImpl is nulled),
    and return NS_BASE_STREAM_CLOSED.
  * Guesses available by doing mBlobImpl->GetSize() which really should be the
    number of bytes available once it's opened, right?
* Read/ReadSegments/Seek/SetEOF():
  * BlockAndWaitForStream()
  * Generate an error for Seek/SetEOF if the stream wasn't seekable.
  * mStream->Read/ReadSegments/Seek/SetEOF(...)
* Tell():
  * If on the owning thread and there's no stream, return 0 because
    invariant-wise that's how it should be.
  * BlockAndWaitForStream()
  * Return error if not seekable, otherwise call Tell()
* Serialize():
  * Return our actor->ParentID() nsID in a RemoteInputStreamParams.
* Deserialize():
  * Throws, comment references InputStreamUtils.cpp, wherein we find that it
    just looks up the existing BlobImpl via BlobParent::GetBlobImplForID(id);
* BlockAndGetInternalStream():
  * BlockAndWaitForStream()
  * returns mStream

### BlobParent::IDTableEntry ###
### BlobParent::OpenStreamRunnable ###

### BlobChild::RemoteBlobImpl ##
isa BlobImplBase, nsIRemoteBlob

### BlobParent::RemoteBlobImpl ###
isa BlobImpl, nsIRemoteBlob

### ??? BlobChild::GetOrCreate ###
Deferred to by BackgroundChild::GetOrCreateActorForBlobImpl and friends.

### ??? BlobParent::GetOrCreate ###
Deferred to by BackgroundParent::GetOrCreateActorForBlobImpl and friends.
