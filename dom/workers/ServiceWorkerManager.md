### ::Register ###

* creates a `cleanedScope`
* AddRegisteringDocument tracks the [doc, scope] association for console
  reporting of failures.
* Creates a new `loadGroup` differing from `docLoadGroup`
* Creates a ServiceWorkerRegisterJob (isa ServiceWorkerUpdateJob) which has the
  cleanedScope on it.

## Jobs ##

* ServiceWorkerJob
  * ServiceWorkerUpdateJob
    * ServiceWorkerRegisterJob

### ServiceWorkerUpdateJob ###
