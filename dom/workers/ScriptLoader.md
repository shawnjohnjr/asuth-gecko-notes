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

### Load ###

Opens the cache with the name of ServiceWorkerPrivate::mInfo::CacheName().  Uses
promise mechanism that ends up in ::ResolvedCallback which then runs through
the list of loaders and invokes (CacheScriptLoader?::)Load on them.

## CacheScriptLoader ##

## ScriptLoaderRunnable ##

## ChannelGetterRunnable ##

Runnable wrapper to invoke ChannelFromScriptURLMainThread used by
ChannelFromScriptURLWorkerThread.

## ScriptExecutorRunnable ##

### PreRun ###
Creates the global scope.
### WorkerRun ###
Loops over the loads, checking for pre-fails for some reason, then sets up the
source and evaluates.
### PostRun ###
Limited error handling.
### ShutdownScriptLoader ###
Cleanup stuff; called by PostRun or Cancel.

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
WorkerPrivate.cpp:CompileDEbuggerScriptRunnable::WorkerRun for debugger scripts.

## Load ##

Thin wrapper around synchronous LoadAllScripts to load an array of URLs and mark
them as not the main script.  Used by
WorkerScope.cpp:WorkerGlobalScope::ImportScripts to load many also by
WorkerScope.cpp:WorkerDebuggerGlobalScope::LoadSubScript to load at most one.
