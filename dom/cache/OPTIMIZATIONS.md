### Have cache.match()ed response.blob() leverage file-backed inputstream ###

As discussed https://github.com/whatwg/fetch/issues/556 and maybe other places,
right now FetchBody<Derived>::BeginConsumeBodyMainThread() just straight-up
uses a MutableBlobStreamListener regardless of the underlying inputstream type.
The ReadStream is handed an IPC-deserialized nsFileInputStream with an FD that
could be wrapped into a FileBlobImpl/subclass.

The trivial issues are:
- ReadStream doesn't expose the underlying stream
- Blobs can be structured cloned whereas Responses can't.  The Cache actors and
  StreamControl infrastructure are not designed to handle the Blob escaping its
  global.  For the life-cycle to work in that case, the Blob really wants to
  imitate how IDB handles things, with the Blob being associated with an actor
  that is known in the parent process.
  - This would imply StreamControl being able to 'upgrade' a stream to a Blob.
