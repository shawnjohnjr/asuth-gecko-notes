====

## e10s

### Work Distribution

Me:
* bug 1231213: allow service worker manager to spawn and hold open a content process
* bug 1231216: allow service worker manager to dispatch an event to a worker in remote child process
Josh
* bug 1231222: perform service worker interception in parent process nsHttpChannel
Andrea:
* bug 1231211: separate service worker controller from docshell
* Bug 1231218: replace nsIDocument in service worker manager with a client interface

### Understandings

See e10s-effort/

### Original Early Bootstrap Discussion

* bug 1182117: Move ServiceWorkerManager to the parent process in e10s

Create WorkerPrivate in another process and hold it alive with a PBackground.
* SWM doesn't have to use it to start with.
* Create infrastructure, write test that let's you do that
* In another bug, update SWM to use it.

Expecting Josh to do:
* nsIIntercept*Channel:
  * 1 for content
  * 1 for parent
* Channel objects are different
  * In parent, nsHTTP channel in single content
  * In child content, need to talk to parent.
* Moving interception to move in the parent side.  So we can have in parent.
* Also move paths to e10s code-paths in necko.

* Splitting out docshell can conflict with Josh.




## No longer deferred: https://github.com/whatwg/fetch/issues/266

Next step:
* Understand the mapping of the spec algorithm onto how it's actually
  implemented.  It's a little scattered and it seems clear the spec has changed
  slightly since the implementation.
  * Special focus on the impact of the worker logic as it relates to clients
    where we're getting the scope from.  I think my understanding of the worker
    scope may be deficient, although I should re-read my pseudocode.  My
    understanding when I was just reading something suggested I have it wrong
    and believe that workers just get their state clobbered to the default
    "no referrer on downgrade" or whatever, but something else contradicted
    that.  Although that something could be only in service workers.
    * => no, this was for local schemes only ("about", "blob", "data", or
      "filesystem").  But now I'm trying to understand where the worker's
      referrer policy is or if it simply does not have one and instead inherits
      and the local kludge is just because it's not expected that the local
      scopes will leak referrers (or perhaps should be forbidden from leaking
      referrers?)
* Make draft changes.
* Investigate the fetch/referrer-policy tests for these.
* Write/modify the tests to make sure there's coverage.


In-flight investigations:
* Is the worker's channel/referrer policy accessed currently?
  * The MainThreadFetchRunnable in Fetch.cpp (which is only used for workers,
    not main thread stuff) does extract the nsIPrincipal and nsILoadGroup from
    the WorkerPrivate in the proxy via GetPrincipal/GetLoadGroup.  (And the
    Fetch is directly invoked from that runnable.)
    * Presumably the main thread bounce and pulling out of the load group at
      that time has to do with rules about touching channels and/or just is a
      design choice to defer pulling the stuff out until they hit the main
      thread since the worker private can already be passed around.  (And maybe
      WorkerPrivate has semantics?)
  * The main-thread variant explicitly sets the mDocument, providing additional
    context beyond the principal and load group.
  * https://bugzilla.mozilla.org/show_bug.cgi?id=1251378 is doing this and is
    in-process.

Sorta answered questions:
* Workers can have their own referrer policy.
  * The header directive explicitly applies based on the Response used to create
    the WorkerGlobalScope.  From the current gecko implementation it seems like
    this is just something that lives on the worker loadInfo's channel and needs
    to be extracted from there.
* The use of "about:client" referrer sentinel is a Gecko normalization of the
  spec.
  * The spec somewhat wackily calls for the string to be "client" internally but
    normalized to be exposed as "about:client" (and there's no way to mutate the
    request's referrer after initialization), so just using about:client is
    about as sane as these things get.
* The existing location for the referrer policy logic is almost certainly the
  right places to make the changes.  Setting the stuff on the channel is the
  only way this conceivably happens.

Answered questions:
* Observability of the mutating steps of the algorithm relative to the referrer
  and referrerPolicy:
  * They are not observable to the caller.  Fetch is defined to use the request
    constructor on its inputs, and the constructor is defined to copy things.
    So anything the algorithm does is not visible on the original request passed
    in.
  * Therefore all that matters is making sure the algorithm is observed by the
    service worker and/or HTTP channel.  Which just means running prior to the
    service worker intercept.


=== Expose ServiceWorker object in DOM object.

bkelly already has some patches up for review:
* https://bugzilla.mozilla.org/show_bug.cgi?id=1263307

=== Expose fragment

Bug: https://bugzilla.mozilla.org/show_bug.cgi?id=1264178
