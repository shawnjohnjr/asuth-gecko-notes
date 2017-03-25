## Further Patch Cleanup Maybe ##

* HttpChannelChild::FinishInterceptedRedirect no longer seems to have a reason
  to exist.  Previously it existed to re-invoke AsyncOpen2/AsyncOpen after the
  redirect.
  * Along those lines, the following state variables that only existed for it
    can be removed:
    * mInterceptingChannel
    * mInterceptedRedirectListener
    * mInterceptedRedirectContext

## Still To Understand ##

Check the *blah* stuff too, but:
* Redirect3Complete's no longer guarded invocation of CleanupRedirectingChannel.

### The redirect and mShouldParentIntercept dance ###

HttpChannelChild::CompleteRedirectSetup had a good comment here.  Basically,

The intercept/suspend life-cycle.  For the parent, grok the suspending.

Context, from nsIRedirectChannelRegistrar,
HttpChannelParentListener::AsyncOnChannelRedirect is likely an initiator.

Parent decides to redirect:
*

Redirect begins in the parent:
* Parent: StartRedirect => SendRedirect1Begin
* Child: RecvRedirect1Begin => Redirect1Event => Redirect1Begin
  * if (mRedirectChannelChild), ConnectParent => SendPHttpChannelConstructor
  * AsyncOnChannelRedirect => eventually
    HttpChannelChild::OnRedirectVerifyCallback
* Parent: NeckoParent::RecvPHttpChannelConstructor => HttpChannelParent::Init =>
  ConnectChannel, which:
  * NS_LinkRedirectChannels digs out the nsHttpChannel, saves as mChannel.  Digs
    out nsINetworkInterceptController, and then HttpChannelParentListener.
    Invokes parentListener->SetupInterceptionAfterRedirect(what was
    mShouldParentIntercept and is locally now "shouldIntercept").
  * In parentListener, if mShouldIntercept, also sets mShouldSuspendIntercept.
    * This will cause HttpChannelParentListener::ChannelIntercepted to save off
      mInterceptedChannel and early return.  Versus handing off the channel
      to FinishSynthesizedResponse.
* Child: HttpChannelChild::OnRedirectVerifyCallback
  * if mRedirectingForSubsequentSynthesizedResponse (triggered by
    FinishSynthesizedResponse calling ForceIntercepted(nsIInputStream*)),
    creates InterceptStreamListener and OverrideRunnable and bails.
  * If didn't bail, ends up calling SendRedirect2Verify.

* Child: OverrideRunnable::Run
  * => Redirect3Complete.
  *

### ChannelEventQueue race rationale ###

The race described is generating an OnDataAvailable when

## Patch Explanation ##

Maybe background:
* Complexity of redirects?  Diversions?
  * Diversions are a means of transferring a child channel back to the parent.
    * Uses:
      * Cert importing initiated in the child. (PSMContentDownloader)
      * ExternalHelperAppChild
  * Redirects and resets:
    * REDIRECT_INTERCEPTION_RESET can cause LOAD_BYPASS_SERVICE_WORKER
  * Redirects:
    * SetupReplacementChannel.  A new channel gets created!

Interception states and/or flow chart?:
* Request w/bypass -> No intercept
* Normal request -> maybe -> ask SWM
  * yes SWM -> dispatch event -> ask SW content code, responds via respondWith
    * yes -> intercepted w/response
    * no -> reset, not intercepted
  * no -> reset, not intercepted

Core explanation thesis:
* Previously, the call to ContinueAsyncOpen() would be deferred until the right
  thing to do was known.
  * AsyncOpen would bail out if it took the interception path and not call
    ContinueAsyncOpen.
  * It would eventually be called if/when:
    * DivertToParent would invoke if mSynthesizedResponse &&
      !RemoteChannelExists().  In this case the actor was actually needed.
    * ResetInterception() is called, since then we're going to the network.
    * OverrideWithSynthesizedResponse is invoked and WillRedirect(aResponseHead)
      (which becomes mResponseHead) is true.  In this case,
      mShouldInterceptSubsequentRedirect also gets set.
* Extra complication on that crazyness in the form of mShouldParentIntercept.
  Need to understand better, but in short, it seems like the parent gets
  involved just so that it can be cut out of the loop again afterwards.
  (In particular, SetupRedirect is the one to call ForceIntercepted(bool, bool)
  which sets mShouldParentIntercept.  It conditionally does that based on
  the mShouldInterceptSubsequentRedirect flag from
  OverrideWithSynthesizedResponse and with explicit mRedirectMode and
  ShouldInterceptURI.  I'm not quite clear

Abandoned explanation theses:
* Previously, AsyncOpen would immediately instantiate an
  InterceptedChannelContent instance and notify the controller that it was being
  controller.  Problem is this actually was the right decision there.  So it's
  got more to do with book-keeping than this.  Possibly with a shift of
  responsibility to the parent.

## New Patch Change Notes ##

### Mechanical ###


NeckoChannelParams.ipdlh *pending*
* removed from HttpChannelOpenArgs:
  * synthesizedResponseHead (OptionalHttpResponseHead): moved to the
    PHttpChannel SynthesizeResponse message.
  * synthesizedSecurityInfoSerialization (nsCString): ditto moved.
  * suspendAfterSynthesizeResponse (bool): *dunno*
* removed from HttpChannelConnectArgs:
  * shouldIntercept (bool)
* added to HttpChannelOpenArgs:
  * redirectMode (uint32_t)
  * isIntercepted (bool)

PHttpChannel.ipdl *pending*
* Redirect2Verify:
  * gained "bool shouldIntercept" argument
* new to-parent messages:
  * SynthesizeResponse(nsHttpResponseHead head, InputStreamParams body,
    nsCString securityInfoSerialization, nsCString finalURLSpec)
    * `head` and `securityInfoSerialization` migrated from HttpChannelOpenArgs.
  * ResetInterception()
* new to-child messages:
  * DispatchFetchEvent()
* removed bi-directional messages:
  * FinishInterceptedRedirect()
    * bi-directional because it's a ping-pong message to ensure processing is
      flushed; child initiates, parent echoes and guarantees not to say more
      stuff, then child deletes itself.

* nsIChannelEventSink.idl
  * added (in my/asuth's reset interception patch) REDIRECT_INTERCEPTION_RESET
    constant which exclusively exists so that
    HttpBaseChannel::SetupReplacementChannel can know to add
    LOAD_BYPASS_SERVICE_WORKER to the new channel's newFlags.

nsIHttpChannelChild.idl *pending*
* removed forceIntercepted(bool postRedirectChannelShouldIntercept,
  bool postRedirectChannelShouldUpgrade)

InterceptedChannel.h *pending*
* InterceptedChannelContent
  * Removed `RefPtr<InterceptStreamListener> mStreamListener` in favor of
    `nsCOMPtr<nsIStorageStream> mStorageStream`.
  * Removed the `InterceptedStreamListener* aListener` argument from
    InterceptedChannelContent's public constructor.

InterceptedChannel.cpp *pending*
* InterceptedChannelChrome
  * ResetIntercept: change: my (asuth) reset interception patch passes the new
    nsIChannelEventSink::REDIRECT_INTERCEPTION_RESET flag to the call to
    StartRedirectChannelToURI.
* InterceptedChannelContent
  * Constructor no longer takes/save aListener consistent with header changes.
  * NotifyController() now creates a storage stream instead of a pipe so that it
    can be serialized. **spinoff and document it**
  * FinishSynthesizedResponse(aFinalURLSpec)
    * pre-patch:
      * Handle URI-ification of aFinalURLSpec and/or secure upgrade of the
        original URI.
      * Compare URIs and if not the same and therefore a redirect, invoke
        mChannel->ForceIntercepted() and BeginNonIPCRedirect().  If the same
        mChannel->OverrideWithSynthesizedResponse().
    * post-patch:
      * If !WillRedirect(mSynthesizedResponseHead), disables conversion via
        mChannel->SetApplyConversion(false).  *why*
      * Unconditionally invokes mChannel->SynthesizeResponse(...)
  * Cancel, pre-patch: null out mStreamListener (now gone)


HttpBaseChannel.cpp *pending*
* removed call to DoNotifyListenerCleanup() from DoNotifyListener().
* SetupReplacementChannel: added (as part of my, asuth's, reset interception
  patch):
  * If redirectFlags included nsIChannelEventSink::REDIRECT_INTERCEPTION_RESET,
    add LOAD_BYPASS_SERVICE_WORKER to newLoadFlags.

nsHttpChannel.cpp *pending*
* ProcessFallback:
  * propagate LOAD_BYPASS_SERVICE_WORKER from newChannel's loadFlags to
    newLoadFlags.
* OpenCacheEntry:
  * the fresh InterceptedChannelChrome's NotifyController method is deferred to
    a subsequent turn of the main thread event loop.  *what thread is
    OpenCacheEntry invoked on?*

HttpChannelChild.h
* HttpChannelChild
  * Add:
    * SynthesizeResponse(aSynthesizedResponseHead, aSynthesizedBody,
      aFinalURLSpec)
    * RecvDispatchFetchEvent
  * Remove:
    * RecvFinishInterceptedRedirect
    * OverrideRunnable
    * ShouldInterceptURI(aURI, aShouldUpgrade)
    * OverrideWithSynthesizedResponse
    * ForceIntercepted
    * mInterceptListener (InterceptStreamListener)
    * mSynthesizedResponsePump (nsInputStreamPump)
    * mSynthesizedInput (nsIInputStream)
    * mSynthesizedStreamLength (int64_t)
    * mSynthesizedResponse
    * mShouldInterceptSubsequentRedirect: Initialized to false, set to true in
      OverrideWithSynthesizedResponse if WillRedirect(mResponseHead), then
      checked in SetupRedirect to trigger ForceIntercepted(false, false) which
      will set mShouldParentIntercept to true.  (And initialize
      mPostRedirectChannelShouldIntercept and mPostRedirectChannelShouldUpgrade
      both to false.)
    * mRedirectingForSubsequentSynthesizedResponse: Initialized to false, was
      set true by ForceIntercepted(nsIInputStream*) signature, checked by the
      OnRedirectVerifyCallback logic.  That gets called by
      InterceptedChannelContent::FinishSynthesizedResponse passing
      mSynthesizedInput which is the input-stream end of the pipe so the
      synthesized response can be read out.
    * mPostRedirectChannelShouldIntercept: was used in AsyncOpen to bypass the
      ShouldInterceptURI check.  Set by HttpChannelChild::ForceIntercepted (now
      gone) which would be set true in HttpChannelChild::SetupRedirect if the
      mode was redirect and ShouldInterceptURI said the redirected URI should
      be intercepted.  ShouldInterceptURI also would have shouldUpgrade mutated
      as a side-effect, which would also be passed to ForceIntercepted to flag
      the mPostRedirectChannelShouldUpgrade variable.
    * mPostRedirectChannelShouldUpgrade
    * mShouldParentIntercept: Initialized to false.  Set to true by
      ForceIntercepted(bool, bool) by SetupRedirect when there's a synthesized
      response that induced a redirect or the channel's redirect mode is manual
      and there's a redirect and it should be intercepted.  Propagated to parent
      in ConnectParent's connectArgs.  Checked by CompleteRedirectSetup
    * mSuspendParentAfterSynthesizeResponse
    * mOverrideRunnable
    * BeginNonIPCRedirect(responseURI, responseHead)
    * InterceptStreamListener: ?fakes out listener?
  * Modified:
    * Redirect3Complete lost its aRunnable argument

HttpChannelChild.cpp
* removed InterceptStreamListener implementation which:
  * takes the nsIStreamListener OnFoo events and calls DoOnFoo on the
    HttpChannelChild owner.
  * Specifically, propagated the following to the HttpChannelChild mOwner:
    OnStartRequest, OnStatus, OnProgress, OnDataAvailable w/synthetic OnStatus
    and OnProgress if the load flags didn't include LOAD_BACKGROUND,
    OnStopRequest w/DoPreOnStopRequest and un-guarded Cleanup().
* HttpChannelChild
  * constructor lost 0-initialized mSynthesizedStreamLength, false-initialized:
    mSynthesizedResponse, mShouldInterceptSubsequentRedirect,
    mRedirectingForSubsequentSynthesizedResponse,
    mPostRedirectChannelShouldIntercept, mPostRedirectChannelShouldUpgrade,
    mShouldParentIntercept, mSuspendParentAfterSynthesizeResponse
  * constructor gained false-initialized mResponseIsSynthesized
  * added RecvDispatchFetchEvent:
    * sets mResponseCouldBeSynthesized = true
    * Gets the nsINetworkInterceptedController, creates an
      InterceptedChannelContent that takes the controller as an arg, and invokes
      intercepted->NotifyController.
* Loses anonymous-namespaced SyntheticDiversionListener is-a nsIStreamListener,
  see DoOnStartRequest deets just below.
* HttpChannelChild
  * DoOnStartRequest no longer creates a SyntheticDiversionListener if
    mSynthesizedResponse (now removed!) was truthy.  The class...
    * Existed for benefit of HttpChannelChild.  Its...
      * OnStopRequest invokes HttpChannelChild::SendDivertOnStopRequest
      * OnDataAvailable reads the data, and sends it up with
        HttpChannelChild::SendDivertOnDataAvailable.
    * In summary, it bounces all the data up to the parent.
  * DoNotifyListenerCleanup emptied out, removing code that would invoke
    mInterceptListener->Cleanup() and null it out if not already null.
  * OverrideRunnable inner class removed.
    * This was instantiated by OnRedirectVerifyCallback if
      mRedirectingForSubsequentSynthesizedResponse.
    * It existed to invoke Redirect3Complete.  Plus error handling if that
      failed due to teardown/reopening, invoking OverrideWithSynthesizedResponse
      to get things going again.
  * RecvFinishInterceptedRedirect removed.  Part of the removed ping-pong IPC
    teardown.
  * SetupRedirect removed the logic that called ForceIntercepted(bool, bool)
    which would set mShouldParentIntercept to true which would cause the
    request to not hit the network in the parent.  The
    mShouldInterceptSubsequentRedirect check would be true if there was a call
    to OverrideWithSynthesizedResponse and WillRedirect(aResponseHead) was true.
    The latter check is on the redirect mode and ShouldInterceptURI(), with the
    check result being saved off.
  * BeginNonIPCRedirect removed.  Its caller was InterceptedChannelContent's
    FinishSynthesizedResponse if the final URL and original did not match.  It:
    * Called SetupRedirect (see above)
    * Directly propagated mSecurityInfo via
      OverrideSecurityInfoForNonIPCRedirect which is just below.
    * Called nsHttpHandler::AsyncOnChannelRedirect with the old channel and
      the new channel and nsIChannelEventSink::REDIRECT_INTERNAL.  That method
      punts over to nsAsyncRedirectVerifyHelper which self-dispatches itself to
      the main thread and will asynchronously do some stuff, especially if
      there's an nsIChannelEventSink.  *is there an nsIChannelEventSink?*  Once
      that's done, the old channel's
      nsIAsyncVerifyRedirectCallback::OnRedirectVerifyCallback gets invoked.
      (Which still exists.)
      * And that used to create an OverrideRunnable.  *FOO*
  * OverrideSecurityInfoForNonIPCRedirect removed, which was called by the
    above now-removed BeginNonIPCRedirect.  It also set
    mResponseCouldBeSynthesized as a side-effect.  (Otherwise that would be
    set in AsyncOpen in the intercept path.)
  * Redirect3Complete(OverrideRunnable* aRunnable) lost its argument and
    the Redirect3Event helper was accordingly updated to not need to pass null.
    Also, minor changes:
    * mOverrideRunnable (removed), mInterceptingChannel (fixup removing) no
      longer set or cleared.
    * The setting case was for mRedirectChannelChild for when, per the comment,
      the "chrome" (really parent?) had been AsyncOpen'ed.
      * mRedirectChannelChild is the replacement channel SetupRedirect created,
        QI'ed to nsIChannelChild.  Its CompleteRedirectSetup method still gets
        invoked.
    * The CleanupRedirectingChannel() call and its comment has been moved out
      from the mRedirectChannelChild QI'd to HttpChannelChild check that
      amounted to: "if mShouldParentIntercept for the redirect, don't clean it
      up yet".  This was why OverrideRunnable existed, *trailed off*
  * CleanupRedirectingChannel list the mInterceptingChannel (removed) Cleanup()
    and nulling.
  * ConnectParent: connectArgs changed to remove mShouldParentIntercept.
  * CompleteRedirectSetup lost its `if (mShouldParentIntercept)` with massive
    comment that explained what was going on.
    * This method is notably only called by Redirect3Complete() if
      (mRedirectChannelChild) which gets setup by SetupRedirect().
      * SetupRedirect is called by both Redirect1Begin and BeginNonIPCRedirect
        (now removed).
    * Removed:
      * Saving from args of mInterceptedRedirectListener,
        mInterceptedRedirectContext.
      * Call to SendFinishInterceptedRedirect() which triggers the IPC channel
        teardown and re-AsyncOpen.
      * A comment that seemed to be requesting the changes that are happening,
        but under-specified.
      * Return to avoid the rest of the logic that
  * OnRedirectVerifyCallback:
    * Removed `if (mRedirectingForSubsequentSynthesizedResponse)` condition
      and logic.
      * Creates InterceptStreamListener and gives it to new OverrideRunnable
        which gets deferred to a future turn of the event loop (it's a dispatch
        to the main thread, but pretty sure we're on the main thread already)
        to invoke Redirect3Complete.  It then bails out, skipping over the call
        to SendRedirect2Verify at the bottom of the method.
    * Added logic to calculate shouldIntercept (if mRedirectChannelChild,
      false otherwise) and pass it up as part of SendRedirect2Verify.
    * This is where my (asuth)'s reset interception patch adds:
      * Extracts shouldIntercept from the redirectedChannel and calls its
        ShouldIntercept(uri) method because it's that channels load flags and
        LOAD_BYPASS_SERVICE_WORKER flag in particular we care about.
  * Cancel: Removed cleanup of mSynthesizedResponsePump (removed) and
    mInterceptListener (removed)
  * Suspend:
    * mInterceptListener (removed)-conditional check removed.
    * mSynthesizedResponsePump's (removed) conditionl Suspend call removed.
  * Resume:
    * mInterceptListener removed from condition.
    * mSynthesizedResponsePump's (removed) conditional Resume call removed.
  * AsyncOpen lost core intercept checking and specialized handling, namely:
    * Decision check on whether intercept can/should happen:
      * mPostRedirectChannelShouldIntercept which got set in SetupRedirect and
        which already ran the ShouldInterceptURI check as part of that change.
      * A call to ShouldInterceptURI is made.
    * Handling:
      * An InterceptStreamListener and InterceptedChannelContent were created,
        with the InterceptStreamListener passed to InterceptedChannelContent to
        listen to its events and call HttpChannelChild->DoOnFoo for each OnFoo
        event received.
      * intercepted->NotifyController() got invoked too which in the normal case
        exists to just call nsINetworkInterceptController::ChannelIntercepted,
        but with error handling to invoke ResetInterception().
      * The InterceptChannelContent instance is now created and
        NotifyController() invoked in RecvDispatchFetchEvent.  There's no
        listener anymore, consistent with changes to InterceptChannelContent.
  * ContinueAsyncOpen (which sends the openArgs to the parent):
    * now propagates redirectMode() from mRedirectMode and isIntercepted() via
      ShouldIntercept(mURI).  ShouldIntercept asks the
      nsINetworkInterceptController which is nsDocShell and which will always
      return true in the child process for navigations, and return true if the
      local ServiceWorkerManager says the document is controlled.
    * removed:
      * mResponseHead handling that initialized the now-removed
        synthesizedResponseHead() and suspendAfterSynthesizeResponse().
      * The synthesizedSecurityInfoSerialization()
  * DivertToParent:
    * removed special-handling where mSynthesizedResponse (now removed) was true
      and !RemoteChannelExists(), setting mSuspendParentAfterSynthesizeResponse
      (now removed), so it could be propagated in the OpenArgs.
  * ResetInterception:
    * Previously, would cleanup mInterceptListener (which propagated events),
      set LOAD_BYPASS_SERVICE_WORKER, and then ContinueAsyncOpen() (which had
      been skipped when AsyncOpen decided to intercept).
  * OverrideWithSynthesizedResponse removed.  It used to:
    * Conditioned on WillRedirect(aResponseHead/mResponseHead):
      * If not redirecting, invoke SetApplyConversion(false) because the
        response is already decoded.  (If redirecting, the result might not be
        intercepted, so the conversion wants to be left intact.)
      * If redirecting:
        * Set mShouldInterceptSubsequentRedirect = true.  This will be
          propagated to mShouldParentIntercept by SetupRedirect calling
          ForceIntercepted(false, false).
        * Call ContinueAsyncOpen()

  * ForceIntercepted(bool, bool)'s mPostRedirectChannelShouldIntercept,
    mPostRedirectChannelShouldUpgrade-setting signature removed.  Also set
    mShouldParentIntercept to true.  This variant was used by SetupRedirect()
    and used to tell the parent to not hit the network. *redirect parent junk*



Still pending notes:
* nsDocShell.cpp
* fetch_tests.js
* HttpChannelChild.cpp
* HttpChannelParent.cpp
* HttpChannelParent.h
* HttpChannelParentListener.cpp
* HttpChannelParentListener.h




## Patch cleanup to-do's ##

### The assumption in HttpChannelParentListener::ResetInterception ###
RESOLVED

There's a paranoia check against mInterceptedChannel.  With a comment wondering
how the reset could happen after ActorDestroy.

Possibilities:
* HttpChannelParentListener::FinishSynthesizeResponse nulls out
  mInterceptedChannel, passing it off to a FinishSynthesizedResponse runnable
  which does NS_DispatchToCurrentThread().
  * Invoked by FinishSynthesizeOnMainThread runnable,
* ClearInterceptedChannel(caller) nulls out mInterceptedChannel if the caller
  lines up.
  * This is invoked by HttpChannelParent::ActorDestroy.

Its ResetInterception happens via (going backwards in time):
* HttpChannelParent::RecvResetInterception invokes the listener's reset.
* HttpChannelChild::ResetInterception invokes SendResetInterception, asserting
  mResponseCouldBeSynthesized and clearing (but not guarding).
* InterceptedChannelContent::ResetInterception() gets called, and checks/sets
  the mClosed guard, invoking HttpChannelChild::ResetInterception.
* One of the following callers is expected:
  * SWP's FetchEventRunnable::ResumeRequest (on main thread).
  * SWP::SendFetchEvent on !registration (race), !mInfo->HandlesFetch(), or in
    the saved-off failRunnable.
  * SWM's ContinueDispatchFetchEventRunnable's HandleError method invokes it if
    the channel is doing funky things.

### HttpChannelParentListener::ClearInterceptedChannel caller-check ###
Context: ClearInterceptedChannel is invoked by HttpChannelParent::ActorDestroy
via `mParentListener->ClearInterceptedChannel(this);`.

Concern: There's a guard on the current mNextListener being the
HttpChannelParent caller and it's not clear why / if there should be.

I put notes in the bug, basically HttpChannelParentListener::OnRedirectResult
synchronously sets the redirected channel to have the same listener while the
old channel's clearing only happens when the actor is destroyed.

### Test issues ###

* Assertion: mSuspendCount > 0, ChannelEventQueue.cpp:67
  * test: bc7: browser/base/content/test/general/browser_bug676619.js
  * analysis:
    * in at least one case, dies when trying to use the id=link2 video.ogg with
      download attribute but no filename.
    * This is the ExternalHelperAppChild::DivertToParent() code path, as
      also indicated by the log entry for SetApplyConversion from that method's
      call to MaybeApplyDecodingForExtension();
* wpt linux opt 2 /fetch/api/redirect/redirect-count.html
  * context
    * "Redirect 308 21 times - Test timed out"
    * manifests locally for me
    * not manifesting on try on linux debug, linux x64 opt, etc.
    * https://treeherder.mozilla.org/#/jobs?repo=try&revision=2159cf9cc333&selectedJob=31653828
  * analysis:
    * This has happened a few times across multiple runs/rebasing.  Assuming
      it's real.  Could be diversion related.

Fixed:
* my revised registration assertion WPT 8 triggering on
  /service-workers/service-worker/update-after-oneday.https.html
  * context:
    * Assertion failure: registration->GetWaiting()->CacheName() == registration->mPropagatingRegistrationCacheName, at /home/worker/workspace/build/src/dom/workers/ServiceWorkerManager.cpp:1732
    * manifesting on try on linux x64 debug, os X 10.10 debug, also on win 7 VM
      debug where it's wpt 9.
  * analysis:
    * the assertion was unsound.  converted to an additional condition check.

## (From Rev1 patch but not obsolete) How stuff works pre-Josh 1231222

### The Players

HSTS (Strict Transport Security) URI Upgrading: STS per spec uses a sticky
header that forces upgrades as long as it hasn't hit zero, so it always needs
be considered, especially when in conjunction with intercept.  (Happily for
sanity, service workers only work on HTTPS!)



InterceptedChannel
* InterceptedChannelChrome: Chrome/parent process half, holds nsHttpChannel.
* InterceptedChannelContent: Content process half, holds HttpChannelChild

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

## Josh's OLD PATCH OBSOLETE Interception changes, bug 1231222 ##

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
