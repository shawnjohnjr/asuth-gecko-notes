This job is the basis for register and update.  Register is covered here too.

## Basic Flow ##

* Update() calls out to serviceWorkerScriptCache::Compare() to determine if the
  SW script (and in the future, its dependencies) have changed byte-wise.

## Detailed Flow ##

* Update()
  * If this was the first register, it's treated as a change.
  * ComparisonResult (implemented via helper CompareCallback class that
    subclasses the serviceWorkerScriptCache's CompareCallback interface class)
    gets invoked with the scope, potentially new cache name, and whether things
    changed.
    * If we got here, it means that the network request happened.  This means
      that the "Service-Worker-Allowed" max scope is fresh, etc.
    * The method validates the max scope against the registration and only then
      considers the aInCacheAndEqual result.
