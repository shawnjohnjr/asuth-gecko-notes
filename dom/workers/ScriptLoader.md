# Overview #

## Service Workers and Cache ##



## Security Things ##

Loads get their exceptions muted if they're not same-origin.  (More specifically
if the worker principal doesn't subsume the channel principal, they get muted.)
Muted exceptions get transformed into network errors.

# Internals #

## GetBaseURI ##

@param aIsMainScript
  Is this the primary script for the worker?  (As opposed to "main" having
  something to do with the main thread.)
@param aWorkerPrivate
  The worker whose base URI is being computed.

Inductive-ish helper to assist in determining the base URI for a worker.

## ChannelFromScriptURL ##

Top level worker load helper.  (nsIDocument* parentDoc)

Q's:
* Confusingly drops parentDoc reference if there's a principal mismatch but
  suggests it's because things are weird for dedicated workers; assuming
  incorrect comment.
* Weird comment reused again below in the inverse case (reaffirming the not
  dropping of the principal.)

Hacks:
* Has hardcoded data URI backwards compat hack.

## ScriptLoadInfo ##

Captures

## CacheCreator ##

Creates a DOM Cache instance in the chrome namespace to stash the service worker
script in.

### CreateCacheStorage ###
@param aPrincipal
  The principal to create a CacheStorage for.

Main-thread method that creates a sandbox via XPC to get its global so it can
instantiate a CacheStorage.  It creates it with a namespace of
CHROME_ONLY_NAMESPACE (and the only other namespace is DEFAULT_NAMESPACE),
presumably to hide the cache from the content.

Duplicate of code in SerivceWorkerScriptCache.cpp or vice-versa.

### Load ###

Opens the cache with the name of ServiceWorkerPrivate::mInfo::CacheName().  Uses
promise mechanism that ends up in ::ResolvedCallback which then runs through
the list of loaders and invokes (CacheScriptLoader?::)Load on them.

## CacheScriptLoader ##
Comes into existence when ScriptLoaderRunnable::RunInternal doesn't
short-circuit into calls to LoadScript because of SW-related flags.  Checks the
Cache for the given script; if it's there, it gets returned.  If it wasn't
there (Match returned undefined), marks mCacheStatus to be "ToBeCached" and
invokes ScriptLoaderRunnable::LoadScript to trigger actual network fetch.

Has concept of mIsWorkerScript propagated from whether the loading script was
the main script.  This only impacts the invocation of (ScriptLoader.cpp's)
GetBaseURI which is all about walking the parent chain for base URI's as
appropriate and is meaningless for SW's.

### Load ###
Core method, triggered by CacheCreator when the cache has been opened
(triggered by a call to CacheCreator::Load by ScriptLoaderRunnable::RunInternal
after 'falling through' to the SW case.)  Is its own promise, does not use
CachePromiseHandler; that's for use when writing to cache.

## ScriptLoaderRunnable ##

* Responsible for setting ScriptLoadInfo::mLoadResult in `LoadingFinished`.
  * Called by OnStreamComplete, CancelMainThread, DataReceivedFromCache,
    RunInternal in the non-SW case (if LoadScript() failed),
    CacheScriptLoader::Fail (possibly via CacheCreator::FailLoaders).
  * For SW's, especially with the cache, this mainly means we expect the

### RunInternal ###
Main decision point between normal worker and ServiceWorker.  Normal quickly
bails to LoadScript.  SW creates a CacheCreator and a bunch of CacheScriptLoader
instances.

### LoadScript ###
Does the actual network load for both normal and serivce workers, with SW's only
getting here if there was a DOM Cache miss.  Behavior diverges for the cases,
with "ToBeCached" (for SW) invoking a "tee" so that both the network-aware
"listener" and the pipe to feed the loadInfo.mCacheReadStream can be fed.  The
SW case also adds the LoaderListener as an nsIRequestObserver to the tee, which
would otherwise only be used as an nsIStreamLoaderObserver (in both cases).  The
nsIRequestObserver gets us the OnStartRequest event which is where the Cache Put
happens.  The nsIStreamLoaderObserver just gets us the OnStreamComplete
notification.

### ExecuteFinishedScripts ###
Finds a contiguous range of fully loaded LoadInfos and passes them to a new
ScriptExecutorRunnable.


### OnStartRequest ###
Creates an InternalResponse to store the script stream, puts it in the cache,
creates a CachePromiseHandler that will mark the load as cached and call
MaybeExecuteFinishedScripts.  Failure deletes the bogus entry.  (Note: race
prone, needs cleanup assistance.)

### OnStreamCompleteInternal ###


## ChannelGetterRunnable ##

Runnable wrapper to invoke ChannelFromScriptURLMainThread used by
ChannelFromScriptURLWorkerThread.

## ScriptExecutorRunnable ##
Used by ScriptLoaderRunnable::ExecuteFinishedScripts to run the individual loads
in batched succession.

### PreRun ###
Creates the global scope.
### WorkerRun ###
Iterates over preceding potential executions.
* Fails-fast if a previous mExecutionResult failed.
For the executions in range:
* Checks for an mLoadResult failure and reports it via
  scriptloader::ReportLoadError and, for top-level scripts, via
  MaybeDispatchLoadFailedRunnable (which consumes the pointer and dispatches, so
  it runs at-most once).
* Sets up the source and evaluates.
  * On error, StealExceptionFromJSContext into mScriptLoader.mRv.
    loadInfo.mExecutionResult is left false.
  * On success, mExecutionResult = true.
### PostRun ###
Limited error handling.
### ShutdownScriptLoader ###
Cleanup stuff; called by PostRun or Cancel.
Handles normalization of mScriptLoader.mRv to deal with muted errors
(transformed into DOM_NETWORK_ERR) or global creation failure/cancellation in
which case it gets to be DOM_INVALID_STATE_ERR.

## LoadAllScripts ##

On-worker-thread creates and synchronously dispatches a ScriptLoaderRunnable

# Public #

## ChannelFromScriptURLMainThread ##

Thin wrapper calling ChannelFromScriptURL.

## ChannelFromScriptURLWorkerThread ##

Creates and synchronously dispatches ChannelGetterRunnable to the main thread.

## ReportLoadError ##

Normalize errors and then ThrowDOMException.

## LoadMainScript ##
Thin wrapper around synchronous LoadAllScripts to load a single script with a
given URL and count it as main.  Used by
WorkerPrivate.cpp:CompileScriptRunnable::WorkerRun for normal scripts and by
WorkerPrivate.cpp:CompileDebuggerScriptRunnable::WorkerRun for debugger scripts.

## Load ##

Thin wrapper around synchronous LoadAllScripts to load an array of URLs and mark
them as not the main script.  Used by
WorkerScope.cpp:WorkerGlobalScope::ImportScripts to load many also by
WorkerScope.cpp:WorkerDebuggerGlobalScope::LoadSubScript to load at most one.
