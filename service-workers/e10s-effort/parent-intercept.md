## How stuff works pre-Josh 1231222

### The Players

HSTS (Strict Transport Security) URI Upgrading: STS per spec uses a sticky
header that forces upgrades as long as it hasn't hit zero, so it always needs
be considered, especially when in conjunction with intercept.  (Happily for
sanity, service workers only work on HTTPS!)



InterceptedChannel
* InterceptedChannelChrome: Chrome/parent process half, holds nsHttpChannel.
* InterceptedChanelContent: Content process half, holds HttpChannelChild

InterceptStreamListener: Content process shim that listens to the
InterceptedChannelContent and

### The Maybe Players

?DivertableChannel: Used by child process to tie off a channel during
OnStartRequest, returning an ChannelDiverterChild IPDL actor that's effectively
a redeemable token that can be handed up to the parent.

### High Level Flow (derived from Static Interaction)

HttpChannelChild gets created.  Its nsINetworkInterceptController is the
nsDocShell.  Its implementation of ShouldPrepareForIntercept and
ChannelIntercepted are straightforward.

HttpChannelParent gets created in the parent, and creates a new
HttpChannelParentListener as its nsINetworkInterceptController (and
NotificationCallbacks).

### Static Interaction (Low Level)

nsINetworkInterceptController
* only exposes 2 methods:
  * shouldPrepareForIntercept
  * channelIntercepted
* implemented by
  * nsDocShell: Directly interfaces with ServiceWorkerManager.
    * shouldPrepareForIntercept is sane:
  * HttpChannelParentListener:

HttpBaseChannel
* GetCallback(nsCOMPtr<T>): NS_QueryNotificationCallbacks-using helper to ferret
  out the interface from:
  * mCallbacks (an nsIInterfaceRequestor, not directly a collection), failing
    over to the load group (if present), asking for its NotificationCallbacks
    (an nsIInterfaceRequestor) too.
* ShouldIntercept: Asks the controller if it should intercept.  :jdm put some
  debug code in there.

HttpChannelParent

HttpChannelParentListener
* mShouldIntercept dictates what ShouldPrepareForIntercept returns.  It gets set
  by:
  * SetupInterception(nsHttpResponseHead) sets mShouldIntercept to true and
    saves off mSynthesizedResponseHead
  * SetupInterceptionAfterRedirect(aShouldIntercept) sets mShouldIntercept and
    if true, sets mShouldSuspendIntercept.  The latter causes ChannelIntercepted
    to alter its behavior to save mInterceptedChannel and bail.
* ChannelIntercepted gets called by InterceptedChannelBase::DoNotifyController,
  called by Chrome/Content::NotifyController.
  * HttpChannelChild::AsyncOpen in a code path :jdm disabled that's if
    ShouldInterceptURI()
  * HttpChannelChild::RecvDispatchFetchEvent, a new code path :jdm added.

HttpChannelChild
* ResetInterruption: ??the fetch decided not to actually intercept?
* OverrideWithSynthesizedResponse: ??got something
* ShouldInterceptURI: factors in HSTS upgrade, then defers to
  HttpBaseChannel::ShouldIntercept.


InterceptedChannel
* InterceptedChannelChrome:
  * Created in OpenCacheEntry
* InterceptedChannelContent:
  * Created in HttpChannelChild::AsyncOpen
    * Intercept if either mPostRedirectChannelShouldIntercept (explicitly set
      via **ForceIntercepted**) or ShouldInterceptURI

ServiceWorkerManager/friends:
* SWM::DispatchFetchEvent: If navigation, uses GetServiceWorkerRegistrationInfo
  to get the registration, creates ContinueDispatchFetchEventRunnable (which
  runs on the same thread and exists in case there is an upload channel which
  may need to asynchronously clone the upload stream, via
  EnsureUploadStreamIsCloneable.)  The CDFER is synchronously invoked if there
  was no upload stream.
* ContinueDispatchFetchEventRunnable::Run makes sure the inner channel didn't
  error out (while messing with the upload stream).  Call mServiceWorkerPrivate
  SendFetchEvent.  Note that the registration we had before gets forgotten
  again; navigation mainly sanity-checking the dispatch and tracking the
  interception for error logging purposes (via AddNavigationInterception).
* SWP::SendFetchEvent: spawns worker if needed, re-gets the SWRI registration,
  creates a keep alive token.  Registration and channel held with
  nsMainThreadPtrHandles which will be handed off.  Creates FetchEventRunnable,
  takes everything.  Calls Init on it, then dispatches it immediately if the
  registration is active, otherwise it queues it in mPendingFunctionalEvents.
BKELLY QUOTE:  `we basically have code to convert the nsIChannel into an InternalRequest... and sicking added InternalRequest to IPC logic in dom/fetch (copied from dom/cache)... so I just need an nsIChannel dummy in my new process and I think it should work` => I think this means we may be able to push the
conversion to an InternalRequest into the parent process and just IPC that
across.
* FetchEventRunnable::Init: (mainthread) gets underlying channel, extracts URI,
  eats fragment to get mSpec.  Pull loadInfo out of underlying channel and
  Referer so that mReferrer, mReferrerPolicy can be saved off.  Pierces
  underlying channel to nsIHttpChannelInternal and also pulls off mRequestMode,
  mRequestRedirect, mCacheMode, mRequestCredentials, and headers using
  VisitNonDefaultRequestHeaders.  Also, if upload channel, clones/processes.
* FER::WorkerRun calls FER::DispatchFetchEvent
* FER::DispatchFetchEvent: Creates InternalRequest, wraps into Request.  Creates
  FetchEvent via its static ::Constructor and then calls PostInit to provide
  mInterceptedChannel, mRegistration, mScriptSpec.  Synchronously
  DispatchDOMEvent.  Some cancellation/resume default processing if respondWith
  not explicitly called.  Let's assume respondWith.
* FetchEvent::RespondWith: Creates RespondWithHandler and hangs it off the
  provided promise.  Also has the FetchEvent depend on that provided promise.
* RespondWithHandler::ResolvedCallback: error checking/handling, security
  checks with error conversion, creates RespondWithClosure, tries to get
  nsIInputStream from InternalResponse via GetUnfilteredBody.
  * If there was a body, get mInterceptedChannel's ResponseBody
    nsIOutputStream, ensures a buffered output stream, then does NS_AsyncCopy
    from the provided inputstream to the outputstream, calling
    RespondWithCopyComplete when done.
  * If there was no body, just directly invoke RespondWithCopyComplete.
* RespondWithHandler::RespondWithCopyComplete: On failure status, creates a
  CancelChannelRunnable, on success a FinishResponse.  Dispatches to main
  thread.
*



e10s-splitup of SWM/friends:
* SWM::DispatchFetchEvent wants to be in the service on the main thread.
*

### Interface Families for context/metadata/etc.

* nsILoadContext (docshell/base): Window context of the load (window, top
  window, top frame, nested frame id), plus additional context
* nsILoadInfo (netwerk/base): Who started the load.
  * Public constructors require either nsINode or nsPIDOMWindowOuter, but friend
    constructor accepts just the window id's.
  * Tracks principals through redirects.

IPC-transport variants:
* SerializedLoadContext: Primarily for low-fidelity nsILoadContext, but also
  can take an nsIChannel and try and get an nsILoadContext via
  NS_QueryNotificationCallbacks with shoddy fallback.  nsIWebSocketChannel does
  the query but without fallback.

## Josh's Interception changes, bug 1231222 ##

### Overview ###

* ServiceWorkerRegistrarParent is introduced as a main-thread repository of
  registration data so that it can be synchronously consulted (IsAvailable).

### Detailed breakdown:

Tracking controlled documents in the parent:
* PBrowser parent can be told that a document is controlled (boolean)
* (TabParent stores mIsControlled, receives the message.)
* SWM in parent process, when MaybeStopControlling, if there's a tabChild, needs
  to tell that to the child.

ServiceWorkerManagerParent changes:
* ServiceWorkerRegistrarParent introduced.

HttpChannelParent:
* ShouldPrepareForIntercept
  * Previously just used mShouldIntercept (?where did this come from?)
  * If mRedirectChannelId, uses nsIRedirectChannelRegistrar to pierce redirect
    to get parent channel so it can use that for the decision.  (??? look into
    impact of redirects on controlled documents more. ???)
    * ??pending speculation: the intercept mechanism uses these redirects?
  * If it's not a subresource and is therefore a navigation, we need to figure
    out if the resulting document should be controlled or not.
    * Pull the origin attributes out of the load context and convert to necko
      origin attributes and figure out the principal the channel will use
      (factoring in all the loadInfo stuff).  Then ask the SWRP if there's
      an interceptor available for the principal and URI.
  * If it is a subresource, then we're going by whether the document is
    controlled or not, so just ask its tab parent.
* FinishSynthesizedResponse: Now gets handed aFinalURLSpec which it propagates
  to the channel, which previously got an empty string.  Comment said the child
  would get the true answer via the redirection notification.

HttpChannelChild:
* SynthesizeResponse

## bkelly's fetchevent-process patch

### Notes from discussion:

* The fetch event life-cycle is too short.  The respondWith can occur
  independent of the waitUntil.  Otherwise the events are similar.
  * :bkelly advocated for using unions over a multiplicity of actors because of
    the boilerplate related to the additional actors.

### Detailed Breakdown

* PContent:
  * Adds "child:" PFetchDispatch.
* FetchDispatchTypes.ipdlh added:
  * FetchDispatchArgs: principal, scope, request, creation time
  * FetchDispatchResult: union of:
    * FetchDispatchSynthesized: IPCInternalResponse response, OptionalIPCStream
      body
    * FetchDispatchReset (empty)
    * FetchDispatchCanceled: nsresult rv
* PFetchDispatch.ipdl added:
  * "parent:" `__delete__(result)`

* ContentParent:
  * Implements {Alloc,Dealloc}PFetchDispatchParent, shunted to
    {Alloc,Dealloc}FetchDispatchParent.
* ContentChild:
  * Implements {Alloc,Dealloc}PFetchDispatchChild shunted to
    {Alloc,Dealloc}FetchDispatchParent, RecvPFetchDispatchConstructor shunted to
    InitFetchDispatchChild.

* ContentProcessManager:
  * Adds GetContentProcessAvoidingId, a helper to say "any content process but
    this one" (that the intercept is coming from).

* ServiceWorkerPrivate:
  * FetchEventRunnable:
    * Add mInternalRequest, extracted from (nsIInterceptedChannel)
      mInterceptedChannel->GetInternalRequest() during Init().  If provided,
      fast-paths out from initializing all the mCacheMode/mRequestMode things.
    * DispatchFetchEvent lightly refactored to use the saved off
      mInternalRequest if available.  Otherwise the existing code that populates
      everything that was extracted out of the channel in Init() is used.

* nsINetworkInterceptController:
  * Expose C++-only stubbed virtual GetInternalRequest/SynthesizeResponse.

* FetchDispatchActors.cpp:
  * (anon) ToInternalRequest(nsIInterceptedChannel) => InternalRequest: does
    what its type says, largely extracted from FetchEventRunnable::Init.
  * RemoteInterceptedChannel : nsIInterceptedChannel: (META: we don't want to
    even have to do these shenanigans going forward, but the
    ConsoleReportCollector may be an issue)
    * ResetInterception, Cancel, SynthesizeResponse all trigger Send__delete__.
      * SynthesizeResponse uses AutoIPCStream.
    * Some getter wrappers ask the mInternalRequest stuff.
    * MOZ_CRASH stubs for most everything else.
  * FetchDispatchChild : PFetchDispatchChild:
    * Init:
      * Unpacks nsIPrincipal from the PrincipalInfo, infers hacky lameScope,
        gets SWRI, uses GetActive(), which will have SendFetchEvent invoked on
        it.
      * Creates InternalRequest from transit form, currently todo on the body,
        stuffs it in a RemoveInterceptedChannel, creates a dummy load group.
  * FetchDispatchParent : PFetchDispatchParent:
    * constructor: takes existing nsIInterceptedChannel.
    * SynthesizedResponse:
      * Takes intercepted channel, extracts inner channel to get at the
        loadInfo for tainting purposes.
      * Otherwise generally calls method on the intercepted channel; wraps in
        a buffer if needed, invokes NS_AsyncCopy, calls
        FinishSynthesizedResponse without waiting for the copy to finish with a
        TODO on it maybe being wrong.
    * `__delete__` handler dispatches to SynthesizeResponse or direct
      mInterceptedChannel->Cancel(...) based on union return type.
  * StartFetchDispatch(ConcentParent, nsIPrincipal, str Scope):
    * Uses PrincipalToPrincipalInfo.
    * news FetchDispatchParent, handing it the nsIInterceptedChannel to hold
      onto for the result.
  * Protocol allocation/deallocation:
    * {Alloc,Dealloc}FetchDispatchChild: new/delete.
    * AllocFetchDispatchParent: return null; StartFetchDispatch is where the
      actual allocation takes place because that's where the arguments are made
      available and the constructor wants the aInterceptedChild.
    * DeallocFetchDispatchParent: delete
    * InitFetchDispatchChild:
      * Casts down to FetchDispatchChild from PFetchDispatchChild, invokes Init.
        Sends `__delete__` in the event of init failure.

* IPCStreamUtils.cpp:
  * SerializeInputStream: Extra handling for the aStream not implementing
    nsIAsyncInputStream, wrapping it into a pipe via NS_NewPipe2 and initiating
    an NS_AsyncCopy, async-ifying the stream.  (NB: I didn't see any obvious
    helpers that duplicate this logic; checked io services and nsIStream* and
    docs in nsStreamUtils.h where NS_AsyncCopy exists.)

* HttpChannelParentListener:
  * Changes from HttpChannelParent::SendDispatchFetchEvent (over IPC) to calling
    the new helper ServiceWorkerDispatchFetch().
    * To facilitate, saves off mServiceWorkerPrincipal (new field) during
      ShouldPrepareForIntercept.

* imgLoader.cpp: ReferrerPolicy clarified to net::ReferrerPolicy (this is the
  triple-represented enum type of horror).  Presumably header inclusion fallout.
