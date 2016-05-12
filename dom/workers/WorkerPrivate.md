# Public #

## WorkerPrivateParent ##

Holds the parent-thread stuff.  WorkerPrivate holds the actual worker thread
stuff.  The two specifically different types allow for clarity in what's being
manipulated.

Presumably, this separation and practical implementation needs compelled the
introduction of ParentAsWorkerPrivate() and WorkerPrivateParent to be a template
so that method could let the parent deal with the concrete class (pointer).


## WorkerPrivate ##

Its members hold the on-worker-thread stuff, its parent class
WorkerPrivateParent holds the things that live on/are owned by the parent
thread.

# Internalish #

## CompileScriptRunnable ##
Runnable that uses scriptloader::LoadMainScript to create the main global and
evaluate the main script into it.
