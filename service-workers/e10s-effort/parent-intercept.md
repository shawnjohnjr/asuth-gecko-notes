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

### Static Interaction

HttpChannelChild
* ResetInterruption: ??the fetch decided not to actually intercept?
* OverrideWithSynthesizedResponse: ??got something

InterceptedChannel
* InterceptedChannelChrome:
  * Created in OpenCacheEntry
* InterceptedChannelContent:
  * Created in HttpChannelChild::AsyncOpen
    * Intercept if either mPostRedirectChannelShouldIntercept (explicitly set
      via **ForceIntercepted**) or ShouldInterceptURI

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
