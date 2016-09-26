Potential/desired cleanups:

## LifeCycleEventCallbacks and other bouncy promise-alikes ##

LifeCycleEventCallback use-case is mainly to get a result set on it, and then
be dispatched back to the main thread.  All while holding a reference to the
method that really wanted the result.

The need to create custom runnables to do this adds a fair amount of boilerplate
and obfuscates call patterns.  Something along the lines of NewRunnableMethod
could be very useful.  Perhaps template magic around native promises so that
the resolved/rejected invocations could be explicitly called out to avoid the
state machine bouncing.
