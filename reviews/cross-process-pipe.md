## File Checklist ##

* [x] AutoUtils.cpp
* [x] AutoUtils.h
* [x] Cache.cpp
* [x] Cache.h
* [x] CacheChild.cpp
* [x] CacheChild.h
* [x] CacheOpChild.cpp
* [x] CacheOpChild.h
* [x] CacheOpParent.cpp
* [x] CacheParent.cpp
* [x] CacheParent.h
* [x] CachePushStreamChild.cpp
* [x] CachePushStreamChild.h
* [x] CachePushStreamParent.cpp
* [x] CachePushStreamParent.h
* [x] CacheStorage.cpp
* [x] CacheStorage.h
* [x] CacheStreamControlChild.cpp
* [x] CacheStreamControlChild.h
* [x] CacheStreamControlParent.cpp
* [x] CacheStreamControlParent.h
* [x] CacheTypes.ipdlh
* [x] DBSchema.cpp
* [x] ManagerId.cpp
* [x] PCache.ipdl
* [x] PCacheOp.ipdl
* [x] PCachePushStream.ipdl
* [x] PCacheStorage.ipdl
* [x] ReadStream.cpp
* [x] ReadStream.h
* [x] StreamControl.h
* [x] TypeUtils.cpp
* [x] TypeUtils.h
* [x] moz.build

## High Level ##

A way to stream data from child process to parent process when all of the data
is not immediately available.  Nicely documented in SendStream.h.

### Invariants ###

Only allowed to create on main thread or worker thread for life-cycle reasons.
There's a comment in SendStream.h that is slightly misleading.  The actor's
life is inherently circumscribed by that of its thread, but the problem is the
associated stream can absolutely outlive the actor.
https://bugzilla.mozilla.org/show_bug.cgi?id=1093357#c35 describes this well.
And since the death of the thread inherently closes the parent actor, this means
that no matter what type of factoring games you play with referenced objects,
we will no longer be able to send any data to the parent once the thread dies.
Since the intuitive/no-surprises life-cycle for a stream is that it is
independent of arbitrary threads, it makes sense to enforce the constraint that
the stream is instantiated on a thread that allows us to maintain stream
semantics.

### Limitations ###

No backpressure.

### Compare/Contrast with SerializeInputStream ###
(InputStreamUtils.h/cpp)

Tagged serialization idiom where stream classes implement
nsIIPCSerializableInputStream and its Serialize method.  Method mutates
passed-in ipc::InputStreamParams and FileDescriptorArray.  Classes like
nsBufferedInputStream know how to propagate to the streams they wrap while also
annotating additional state like bufferSize.

## Details ##

### SendStreamChildImpl ###

Uses SendStreamChildImpl::Callback to bounce OnInputStreamReady events to the
thread where the actor lives, where the runnable invokes OnStreamReady.  Keeps
creating and nuking the runnables; presumably doesn't reuse for state machine
clarity, but no real benefit.  When invoking AsyncWait it specifies no event
target so the bouncing is somewhat gratuitous, although perhaps there's
something I'm missing.

Worker feature that returns true to "keep the worker thread alive until the
stream is finished."

Uses 32k max read/IPC message size with batching rationale.

### SendStreamParentImpl ###
