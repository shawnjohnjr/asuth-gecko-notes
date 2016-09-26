Understands how to fetch the root SW script and check the existing cache,
bytewise compare them, then invoke callback.ComparisonResult() with the new
cache name and scope if there was a difference.

Has nothing to do with imported scripts and getting them into the cache (at this
time.)

## Coming Changes ##

https://bugzilla.mozilla.org/show_bug.cgi?id=1290951 will have us do a
byte-for-byte check of all imported scripts as well.  This will be accomplished
by enumerating the contents of the cache and re-fetching each (original,
pre-redirect presumably) request and doing a byte-wise comparison on those.  We
can stop as soon as we find any changes.

## Implementation ##

### Classes ###

* CompareManager: Drives everything.

* CompareNetwork: Network helper, Fetches the (root) SW script over the network
  into an nsString exposed via Buffer(), calls CompareManager::NetworkFinished
  on completion and also metadata methods as it goes (ex: SetMaxScope on
  "Service-Worker-Allowed" header).
  * In charge of the specifics of the fetch: no redirects allowed,
    "Service-Worker: script" header and related result enforcement like MIME
    type.
* CompareCache: Cache helper, state machine to open cache, Match on the (root)
  SW script, then stream its body into the buffer, calls CacheFinished on
  completion.


### Compare() tracing ###

linearized order.

Compare
* Instantiates CompareManager, calls Initialize.

CompareManager::Initialize
* Gets a CacheStorage instance by creating an XPConnect sandbox and bypassing
  secure context checks.
* Issues a network fetch of the root script via CompareNetwork.
* Behavior varies based on aCacheName.  If provided, a CompareCache is created
  to load the script from the cache.
* CompareNetwork/CompareCache will call into NetworkFinished/CacheFinished
  respectively, with calls ending up in ComparisonFinished.

CompareManager::ComparisonFinished(aIsEqual)
* aIsEqual gets to be false if there was no aCacheName originally or if there
  was a byte-wise difference of the two scripts.  NB: This all will end up
  changing a bit when we do.
* Calls WriteNetworkBufferToNewCache.  Creates a new cache name because every
  different version of the SW gets its own cache.
* Does a little state-machine dance since the CacheStorage::Open call returns a
  promise, which ends up in a call to ::WriteToCache() which then invokes the
  originally passed-in mCallback.
