
## Theory Pieces ##

### General families ###
* A race related to an edge-case involved with a redirect.
* Edge-case involving a tracking blocker/other extension meddling.  This seems
  very possible given the low incidence rate since extensions don't get a chance
  to meddle in Service Worker interception *unless* there's a redirect involved.
  * The crash reports do indicate a high level of extensions, some of which
    explicitly are involved in blocking.  However, there's a lot of UUID-named
    extensions which I didn't perform lookups on.
* Repeated/contradictory state issued from child to parent that should have been
  guarded against.  (Although these crashes also happened in 56a1 and we added
  new guards in 55a and even before related to serviceworker responses.)

* Suspend results in mCallOnResume saving off Async* functions to be invoked.

## State Understanding ##

We have an initial channel that hit redirect state and its in-progress redirect
successor, which we've told the child process about.  It's in the process of
"connecting" a new HttpChannelParent to the new/resulting channel which has not
yet progressed to the AsyncOpen() stage.

### Specific member values expected

nsHttpChannel:
* mWaitingForRedirectCallback = true.  (Because we're in the async redirect
  verify process that's currently waiting on the child.)

### Why ###

* We had an initial channel created, including a parent listener.
* A redirect was initiated, getting far enough to send redirect1 to the child
  so that it can also sign off on the redirect.
* We did not progress to actually AsyncOpen()ing the redirected channel.
  * We know this because Redirect1Begin only initiates the child's
    gHttpHandler->AsyncOnChannelRedirect call that will validate in the child
    immediately after invoking ConnectParent.  That process will culminate in
    HttpChannelChild::OnRedirectVerifyCallback() being invoked which will
    call SendRedirect2Verify which then invokes OnRedirectVerifyCallback which
    is what would let the parent progress to invoking AsyncOpen().

### Potential Races ###

The Redirect1Begin/ConnectParent is a loop initiated from the parent to the
child that goes back to the parent.  This should normally be tight, but one
might presume that it would be possible for the child to enter/issue a state
change to the parent between when it issues the SendRedirect1Begin call in
StartRedirect and when the connect comes in.

## Specific Theories ##

**HERE**
* Could this just be a case of a canceled channel?
  * ??? HandleAsyncRedirect will check mStatus and divert over to
    ContinueHandleAsyncRedirect(rv) with the failure, and either way it will
    invoke DoNotifyListener().
    * ??? HandleAsyncRedirect seems to come from nsHttpChannel::ReadFromCache on
      WillRedirect(mResponseHead) which is not a code path we'd be using.
  * HttpChannelChild will straight up do SendCancel().  The Parent will bounce
    it over directly to the nsHttpChannel.
  * nsHttpChannel::Cancel() does have a LOG() warning if
    mWaitingForRedirectCallback, but that's it.
    * That gets set by WaitForRedirectCallback which was called by
      StartRedirectChannelToURI right after gHttpHandler->AsyncOnChannelRedirect
      check was kicked off.
* ???
  * No?  In a redirect, the replacement channel propagates over mCallbacks
    during SetupReplacementChannel

ooo, in StartRedirectChannelToURI we missed the mInterceptCache == INTERCEPTED
case directly invokes ForceIntercepted...?

??? In ContinueProcessResponse2, under the redirect status switch, if
AsyncProcessRedirection fails and DoNotRender3xxBody(rv) returns true, we invoke
DoNotifyListener().

## Overview ##
Crash stack is:
* HttpChannelParent::ConnectChannel(unsigned int const&, bool const&)
* HttpChannelParent::Init(mozilla::net::HttpChannelCreationArgs const&)
* NeckoParent::RecvPHttpChannelConstructor

The failure is in the notification callbacks lookup:
```
  nsCOMPtr<nsINetworkInterceptController> controller;
  NS_QueryNotificationCallbacks(channel, controller);
  RefPtr<HttpChannelParentListener> parentListener = do_QueryObject(controller);
  MOZ_ASSERT(parentListener);
  parentListener->SetupInterceptionAfterRedirect(shouldIntercept);
```

See "How do we get into crash state?" below for the causality trace, but the
basic scenario here is:
* In the parent, HttpChannelParentListener::AsyncOnChannelRedirect got told
  about the old channel and the new channel.  It:
  * Registered the new channel with the nsIRedirectchannelRegistrar service,
    which handed back the mRedirectChannelId.
  * QI'd its mNextListener (currently the HttpChannelParent; this only changes
    during diversion), thereby invoking HttpChannelParent::StartRedirect, which
    sends the id to the child.
* The child receives the id.  It creates a new HttpChannelChild via
  SendPHttpChannelConstructor using HttpChannelConnectArgs.
* Back in the parent, we get our above stack where we're fishing out the newly
  created nsHttpChannel and we want to propagate the shouldIntercept argument
  from the HttpChannelConnectArgs.
* We end up with a null parentListener, which means that either
  NS_QueryNotificationCallbacks for an nsINetworkInterceptController returned
  null or do_QueryObject failed to return an HttpChannelParentListener.  (The
  latter uses nsGetInterface template magic to extract the IID.
  HttpChannelParentListener explicitly does
  `NS_DECLARE_STATIC_IID_ACCESSOR(HTTP_CHANNEL_PARENT_LISTENER_IID)`.)

Of particular emphasis is that this is the *new* channel that lacks the
controller which means that any state mutations occuring on the *old* channel
need to have occurred prior to the SetupReplacementChannel call.  This rules
out a whole bunch of races, especially because the child is completely unable
to reference the new channel yet; that's what this whole connect state is about.

### How do Notifications Get Set? ###

Notification callbacks should initially get set in
HttpChannelParent::DoAsyncOpen with a freshly created HttpChannelParentListener.
Its invocation stack/causality chain looks like (this is going backwards in
time!):
* (Parent Process)
  * (HttpChannelParent::DoAsyncOpen)
  * HttpChannelParent::Init(w/union type HttpChannelOpenArgs)
  * NeckoParent::RecvPHttpChannelConstructor
* Child Process
  * HttpChannelChild::ContinueAsyncOpen
  * VARIES, but since we know we're crashing in a redirect, we can eliminate
    some possibilities.
    * HttpChannelChild::ResetInterception().  This happens because we need to go
      to the parent because we're resetting interception.
    * HttpChannelChild::OverrideWithSynthesizedResponse() when
      nsHttpChannel::WillRedirect(mResponseHead).  This happens because
      redirects currently require going to the parent.
    * HttpChannelChild::AsyncOpen() when not intercepting.
    * NOT THIS: HttpChannelChild::DivertToParent().  Iff there's a synthesized
      response and no parent actor yet.  This happens to cause the parent actor
      to be created.

We say initially because HttpBaseChannel::SetupReplacementChannel in the
redirect case will propagate mCallbacks across to the newChannel.  This can
get called from (all nsHttpChannel::):
* _StartRedirectChannelToURI_: used by InterceptedChannelChrome and
  HttpChannelParent::DoAsyncOpen() off of aAPIRedirectToURI
  * InterceptedChannelChrome::{ResetInterception,FinishSynthesizedResponse}:
  * HttpChannelParent::DoAsyncOpen() gets aAPIRedirectToURI from the child's
    mAPIRedirectToURI set in HttpBaseChannel::RedirectTo, which is expossed by
    HttpChannelChild's subclassing it.
  * ContinueProcessResponse2: _304_ status switch case
* ContinueProcessRedirectionAfterFallback
  * AsyncProcessRedirection
    * ContinueProcessResponse2: _300,301,302,303,307,308_: switches based on
      httpStatus from mResponseHead->Status()
* ProcessFallback: App-cache stuff, ideally not this?
* AsyncDoReplaceWithProxy: Proxy stuff, ideally not this?

### How do we get into crash state? ###

So the above happens first.  Then we get the ConnectChannel case, whose
causality looks like (going backwards in time):
* (Parent Process)
  * (HttpChannelParent::ConnectChannel)
  * HttpChannelParent::Init(w/union type HttpChannelConnectArgs)
  * NeckoParent::RecvPHttpChannelConstructor
* Child Process
  * HttpChannelChild::ConnectParent
  * HttpChannelChild::Redirect1Begin
  * RecvRedirect1Begin/Redirect1Event; this is the ChannelEventQueue abstraction
* Parent Process
  * HttpChannelParent::StartRedirect
  * HttpChannelParentListener::AsyncOnChannelRedirect (via mNextListener)

#### What would have continued to happen if we didn't crash ####

(down is going forward in time)
* Child Process
  * HttpChannelChild::Redirect1Begin regains control from ConnectParent...
  * nsHttpHandler::AsyncOnChannelRedirect
  * (nsAsyncRedirectVerifyHelper asks event sinks and others for what they know,
     culminating in the call to the next thing.)
  * HttpChannelChild::OnRedirectVerifyCallback
  * HttpChannelChild::SendRedirect2Verify
* Parent Process
  * HttpChannelParent::RecvRedirect2Verify
  * HttpChannelParent::ContinueVerification
  * HttpChannelParent as nsIAsyncVerifyRedirectCallback::ReadyToVerify: I think?
    The general call graph here should net out the same regardless.
  * HttpChannelParent::ContinueRedirect2Verify
  * nsIAsyncVerifyRedirectCallback::OnRedirectVerifyCallback to
    nsAsyncRedirectVerifyHelper (probably)
  * nsHttpChannel as nsIAsyncVerifyRedirectCallback::OnRedirectVerifyCallback,
    which then pops the mRedirectFuncStack contents to call...
  * nsHttpChannel::ContinueAsyncRedirectChannelToURI

### How might notifications not get set? ###

As noted in "How do Notifications Get Set?" we're dealing with the new channel
which receives its callbacks in the old channel's
HttpBaseChannel::SetupReplacementChannel where mCallbacks is propagated via
SetNotificationCallbacks.

This would imply a sequence (time going forward here):
* ReleaseListeners/DoNotifyListener happens
* StartRedirectChannelToURI happens, invoking SetupReplacementChannel

### How do notifications get cleared? ###

HttpBaseChannel::ReleaseListeners clears mCallbacks, which is where it's stored.
Callers:
* HttpBaseChannel::DoNotifyListener()
  * AsyncAbort() via HttpAsyncAborter<T>::HandleAsyncAbort() template magic
    (nsHttpChannel subclasses/instantiates it with glue via ::HandleAsyncAbort).
    This appears to be only invoked after AsyncOpen completes.
  * (all others here are nsHttpChannel)
  * ContinueHandleAsyncRedirect (twice).  ContinueHandleAsyncRedirect is only
    triggered by HandleAsyncRedirect, and that appears to only be triggered by
    a cache entry telling us to redirect.
  * HandleAsyncNotModified
  * ContinueHandleAsyncFallback
  * ContinueProcess{Normal,Response2,Response3}
  * ContinueAsyncRedirectChannelToURI: Happens after nsAsyncRedirectVerifyHelper
    completes, which is not yet the case in this situation.
* nsHttpChannel::OpenRedirectChannel() after mRedirectChannel->AsyncOpen{2} is
  invoked, passing over the mListener.  Note that it will fail to invoke
  ReleaseListeners() if AysncOpen fails.
* nsHttpChannel::ContinueDoReplaceWithProxy: same deal, after AsyncOpen{2}.
* nsHttpChannel::ContinueProcessFallback: same deal again
* nsHttpChannel::ContinueProcessRedirection: same deal again
* nsHttpChannel::AsyncOpen() if NS_CheckPortSafety(mURI) fails.
* nsHttpChannel::AsyncOpen2() if doContentSecurityCheck fails.
* nsHttpChannel::OnStopRequest() at the bottom

Because we're not to the AsyncOpen{2} stage yet, we expect DoNotifyListener is
the source of our release call for the crash.
