## Read First ##
(Theory: we can help people by pointing them at the most relevant documentation
in a component to help them get an overview.  These simple pointers are
something that take a sufficiently minimal amount of effort that they can be
kept up-to-date.  Open questions are how to accomplish this with minimal
hurdles, etc.)

* Manager.h

## Infographic Ideas ##

Nested box sort of thing for high-level operations showing control flow with
the outermost boxes representing files and inner boxes representing
classes/functions, possibly with mini-relationships like ReadStream's paired
interfaces.

Ex: Follow what happens with a match...
* Manager::CacheMatchAction
* db::CacheMatch
* FileUtils.cpp BodyOpen


## Basics ##

Streams everywhere, no blobs.

## IPDL ##

Protocols:
* PCacheStorage: Handle to the per-principal cache root that conceptually owns
  the individual caches, used to send cache operations to the parent.  The
  protocol doesn't name-check PCache because all of the work/etc. happens in the
  PCacheOp's (which are the same type for both PCacheStorage and PCache).
* PCache: Handle to a cache (the specific granularity of storage), used to send
  cache operations to the parent.
* PCacheOp: Requests against a PCache or PCacheStorage; deleted with error code
  and result.
* PCacheStreamControl:
  * Can tell parent that the stream got closed
  * Can tell the child to close a specific stream or all streams.
  * ?? so, high level, we can have a closure because the source of the stream
    died or because the sink/consumer no longer cares.

Types (CacheTypes.ipdlh):
* Spec stuff:
  * CacheQueryParams
  * CacheOpArgs: discriminated union, all kids have naming scheme of blahArgs
  * CacheOpResult: discriminated union, all kids have blahResult scheme
* Other internal gunk
  * CacheReadStream:
    * id:
    * PCacheStreamControl
    * IPCStream stream: Provides the control

## Classes / File Clusters ##

DOM exposed bindings:
* Cache

Parent stuff:
* Manager related
  * Manager::Factory: Singleton
  * Manager
  * ManagerId: Equivalence abstraction for managers; tuples principal and
    (quota) origin, but only actually uses the quota origin for tests.  With
    the context/container work, this may need to change to mainly compare
    principals, but it's already abstracted, so, win.
  * Manager::Listener
  * Manager.cpp:
* Protocol implementations
  * CacheParent
  * CacheOpParent
* I/O stuff:
  * Context: RAII mechanism that controls QuotaManager lifecycle, mediates I/O.
    * Context::Activity: Interface class allowing in-progress stuff to register
      with the context so it can cancel them if it gets canceled.
  * DBAction: Base classes for the actual actions that live in Manager.  they
    just have the magic to open the given connection.
  * DBSchema: collection of actual database manipulating functions.
* Stream stuff:
  * StreamConrol family:
    * StreamControl
    * CacheStreamControlParent
    * CacheStreamControlChild
  * StreamList:
  * ReadStream family:
    * ReadStream the outer class is just a thin shell that implements
      nsIInputStream and dispatches to ReadStream::Inner for all the actual
      work.  By existing it is able to have an IID so that instances of it can
      be detected, although I haven't found if/where that's used yet.
    * ReadStream::Controllable: the cache internal (public) interface for
      controlling the stream which is then implemented by ReadStream::Inner.
    * ReadStream::Inner:

Child stuff:
* Protocol implementations
  * CacheChild
  * CacheOpChild

## Manager ##

?Thread-usage: Manager docs say the manager should be used on the thread it was
created on.  Also that there's only one manager.  Presumably the former should
just more tightly describe that this thread must be PBackground? => Yeah,
Manager::Factory::GetOrCreate assets it's on the background thread.

## Streams ##

### Where Do They Come From, Where Do They Go, Cotton Eye Joe? ###

Sources:
* Fetch Response: The Response constructor normalizes everything.  Given a body
  that could be tons of different things it creates an nsIInputStream.  Most of
  these are normal Necko things:
  * ArrayBuffer, ArrayBufferView: NS_NewByteInputStream on the buffer data.
  * Blob: BlobImpl->GetInternalStream()
    * which for the core types ends up being
      * BlobImplFile: NS_NewLocalFileInputStream or
        NS_NewPartialLocalFileInputStream
      * BlobImplTemporaryBlob: nsTemporaryFileInputStream
      * EmptyBlobImpl: NS_NewCStringInputStream
      * BlobImplMemory: DataOwnerAdapter::Create to NS_NewByteInputStream.  The
        DataOwnerAdapter seems to be a life-cycle hack but its hack comment
        seems misleading/out-of-date.
  * FormData: Serializes to send-rep I guess using GetSendInfo
  * USVString: Encodes to UTF-8, slaps on the text/plain;utf-8 mime-type and
    does NS_NewCStringInputStream.
  * URLSearchParams: NS_NewCStringInputStream with the form encoded mimetype.
* DOM Cache: The match command produces a SavedResponse (from SavedTypes.h)
  which wraps the ipdlh CacheResponse type.  FileUtils.cpp:BodyOpen is then
  invoked which does FileInputStream::Create which immediately opens the
  file (the FileInputStream is a QuotaManager thing that extends
  nsFileInputStream).
  * The resulting input stream with the nsID mBodyId is added to the passed-in
    StreamList.  Similarly, in a matchall, it's just that multiple of these are
    added to the StreamList.
  * The StreamList with its input streams remains paired with the mResponse or
    mSavedResponses in the all case.
  * These paired representations of course happen in the parent process where
    the database and such live.  CacheOpParent::OnOpComplete invokes
    AutoParentOpResult::Add (possibly multiple times).  The CacheResponse
    gets propagated in, but then the stream magic happens in
    SerializeReadStream:
    * The actual file stream gets pulled out of the stream list.
    * A CacheStreamControlParent is created and the PBackground manager told to
      instantiate a matching child.  (This is created/cached as mStreamControl.)
    * The StreamList gets SetStreamControl invoked with that control parent.
    * A ReadStream is created.  The Inner is made explicitly aware of the
      control parent.
    * The ReadStream is told to serialize itself into the CacheResponse's
      CacheReadStream "body".  The id gets serialized, the reference to the
      PCacheStreamControl/CacheStreamControlParent, and an AutoIPCStream is
      created to serialize things.  The AutoIPCStream is also moved into the
      UniquePtr cleanup list.  The cleanup list might be a slight misnomer now
      with the RAII AutoIPCStream stuff since it's mainly a question of making
      sure TakeValue() gets invoked to stop the destructor from nuking the
      resources.  The ReadStream::Inner then also forgets the stream so it
      doesn't end up getting closed via that avenue either.
    * Q: The involvement of ReadStream here seems odd unless something smarter
      happens in the same-process case; does that happen?  Where?  If not, it's
      probably planned to happen.  What's the bug?
  * (prior to calling OnOpComplete, StreamList::Activate gets invoked which
     causes it to AddRefCacheId and AddStreamList and AddRefBodyId on the
     Manager which helps keep it alive.)
  * OnOpComplete does Send__delete() invoking SendAsOpResult() on the auto
    result (per the protocol where the cache op deletes with the result).  It
    mainly just invokes TakeValue on everything in the cleanup list.


### Questions to Answer ###

#### Why does ReadStream need to exist? ####

Comments indicate it's to know when things can be deleted / other optimizations.
IndexedDB already has some smarts with its reference-counted, file-backed blobs,
so the Q is why can't we just be using that?  The answer is the IndexedDB
solution is very IndexedDB-specific and has cleverness related to de-duplication
of the same Blob sent over the wire in a request, etc.

MOOT, but need to extract better definition of how CacheReadStream works:

First guess is that IndexedDB's modeling of things as an
already-there-in-its-entirety Blob is problematic.  But prior investigations
have suggested that our Cache implementation has no affordance for the Cache
persisting and exposing still-downloading streams.  Perhaps the Cache, however,
handles morphing the stream so that the same object reference pre-persistence
also corresponds to the post-persistence data.  In IndexedDB although I've
proposed a magic morphing implementation detail, this is not the case and the
only way to get the file-backed Blob is to issue a read following a write.

Need to understand management life-cycle.  CacheReadStream combines the id with
the PCacheStreamControl with the IPCStream.  The IPCStream notably can (now)
end up as a PSendStream or as an atomically serialized representation.  While
the PSendStream does have some baked-in life-cycle stuff, the serialized
representations do not.  So PCacheStreamControl could matter there.  Q of
whether it could correspond to an outstanding refcount, etc.

## TypeUtils ##

A mix-in that provides transformation methods from the various request/response
representations used by the cache.  Subclassed by Cache, CacheOpChild, and
CacheStorage.  While effectively a convenience for cleaner code, it does depend
on subclasses to implement GetGlobalObject().  The type is also directly used
by the AutoUtils helpers/wrappers.

## Stuff that could be refactored out ##

* Context: It's totes abstracted for QuotaManager interaction.
* Feature: Generic thing that seems to just spread the feature notification over
  a list of actors that's dynamically updated as stuff happens.
* DBAction's database query URL generation.
