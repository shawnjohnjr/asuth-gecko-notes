Meta: Bug 1231222 overhauls how Service Worker (spec, MDN: overview details) network interception happens in Gecko.  I'm writing this post in support of the review process and to help provide some point-in-time higher level documentation.

Disclaimers: Examples involving Service Workers are intentionally simplified unless relevant to the parent intercept changes.  All code links are made to DXR at a file granularity because I expect those links to be long-term stable and less of a hassle than linking to a query page.  You may need to control-f find the term I refer to, however.  For code exploration and understanding, I recommend searchfox.
Background (which you may already know)
Service Workers
Service Workers and Fetch

Service Workers that have registered a fetch (spec) event handler during initial script evaluation will receive fetch events.  These fetch events fall into 2 groups:

    Navigation: A new document/worker "client" is being loaded and the URL matches the registration's scope URL.  If the SW responds, the registration is said to control the client and will receive fetch events for all future requests.  If the SW does not respond, then the request will go to the network/HTTP cache as if the SW did not exist and the SW will not receive fetch events for sub-resources loaded by the resulting page.
    Sub-resources:  A controlled document/worker is making a network request which can be to the page's own origin or any other origin.  The SW has an opportunity to respond to the fetch request.  If the SW does not respond, then the request will go to the network/HTTP cache as if the SW did not exist.  The SW will continue to receive fetch events for controlled pages even if it never responds to any of them.

"fetch" event handlers must do one of the following things when they receive their event:

    Call respondWith() before returning, providing a promise that must resolve to a Response object.  If the promise is rejected or is resolved with anything other than a Response object, the fetch's result will be a network error.  In other words, once respondWith() has been invoked, the Service Worker has committed to a response and will no longer automatically fall back to contacting the network.  However, the SW will can still call fetch(request) itself to approximate the same result.
    Call preventDefault() without calling respondWith(), resulting in a network error for the fetch.
    Don't call respondWith() or preventDefault(), resulting in the fetch going to the network like the Service Worker did not exist.

Service Workers and Multiple Processes: Now

ServiceWorkerManager is the brains of Gecko's Service Worker implementation.  It knows if a given URI is covered by a Service Worker scope.  It knows if a given document is controlled.  It lets you dispatch fetch events, notification events, everything.  Currently there's one ServiceWorkerManager per process.  The ServiceWorkerManagers share unified race-proof Cache API storage in the parent process, and a race-prone-broadcast understanding of what registrations exist and which chrome-namespaced cache name where the SW script and its dependencies are stored.

Diagram showing one ServiceWorkerManager per process.

This will change in the near future, but the net result is that for the patch under discussion:

    Each content process has its own ServiceWorkerManager.
    When a ServiceWorker instance needs to be spun up, it will be spun up in the current process.  It does not matter if there is already an equivalent instance alive in another content process.
    When fetch events get dispatched, they will be dispatched in the current process and the results produced in the current process.

Terminology note: When I say "instance" I refer to a distinct version of the ServiceWorker as identified by its id issued during the update process with its own specific Cache API storage. In other words, each of a ServiceWorkerRegistration's installing, waiting, and active is a different instance.
Service Workers and Multiple Processes: The Future

Part of why we need the parent intercept patch is for that bright future where there is only one ServiceWorkerManager and it lives in the parent process.  And there is only ever one ServiceWorker instance across all content processes in the browser.  To start with, all of these instances will live in a single Service Worker process for main thread contention reasons.  As various Project Quantum advancements are made that reduce main thread contention, we should be able to spawn ServiceWorkers in normal content processes.

Simplified diagram of the beautiful future where there's only one ServiceWorkerManager.

The diagram above attempts to capture a very simplified version of the expected first steps.
Necko HTTP Channels
Basics: URIs, Channels, Stream Listeners

In Gecko and its network layer, Necko, channels are the primary abstraction for network requests.  If you have a string URL, you ask nsIIOService to create an nsIURI and using that, create an nsIChannel (which is also an nsIRequest).  nsIChannel's asyncOpen2 method begins the asynchronous process of opening the channel, with notifications provided to the passed nsIStreamListener/nsIRequestObserver "listener" which exposes the nsIInputStream which contains the actual data stream.

Each channel corresponds to exactly one URI, exposed as "URI" on the nsIChannel.  In the event of a redirection, a new channel is created with the new URI.  The original URI is propagated to the new channel as well, and is exposed as "originalURI".  HTTP channels additionally may store the "documentURI" of the document originating the request which allegedly is for 3rd-party cookie blocking, but the mozIThirdPartyUtil implementation and most other code now gets their information from the channel's associated nsILoadInfo.  nsILoadInfo provides important context about why a network load was started and what it will be used for.

Each network protocol has its own implementation directory beneath mozilla-central/netwerk/protocol.  Service Workers only support the HTTP protocol (in secure contexts).
HTTP Channels under Multi-process Firefox (AKA Electrolysis AKA e10s)

Simplified HTTP Channel class diagram.

In single process Firefox (or for HTTP connections initiated from the parent process), an instance of the "real" HTTP channel class nsHttpChannel is created.  In multi-process Firefox when code in a child (AKA content) process wants to open an HTTP(S) URI, an HttpChannelChild instance is created instead.  Each HttpChannelChild instance *may* cause a corresponding HttpChannelParent instance to be created in the parent process.  The child and parent communicate with each other via the PHttpChannel IPDL protocol.  I place an emphasis on *may* because that is something this patch changes.

The HttpChannelParent instance creates and owns an nsHttpChannel instance to do the "real" work.  nsHttpChannel deals with the HTTP cache and perform the actual network communication.

HttpChannelParent also creates an HttpChannelParentListener which is the nsIStreamListener it passes to the nsHttpChannel.  The HttpChannelParentListener provides a layer of indirection so that the stream can be "diverted" from the child back to the parent and the HttpChannelParent can be removed from the picture.  In the simplest case, the HttpChannelParentListener simply redirects OnStartRequest, OnDataAvailable, and OnStopRequest callbacks to its mNextListener, the HttpChannelParent.

HttpChannelChild and HttpChannelParent work together to proxy requests from the child to the parent and data from the parent to the child.  Data travels as nsCStrings in OnTransportAndData IPC messages, not via file descriptors passed to the child process.

Logic in the child process doesn't notice the difference between HttpChannelChild and nsHttpChannel because it interacts via XPCOM interfaces like nsIHttpChannel rather than the concrete implementation types.  This is helped by nsHttpChannel and HttpChannelChild sharing HttpBaseChannel as a base class that contains common logic related to implementing the XPCOM interfaces.
Complication: Redirects

As noted above, each channel is associated with a single URI.  Accordingly, a new URI means a new channel.  For HTTP, redirects usually happen because an HTTP request returned a 3xx response.  However, Service Workers introduce an additional complication because the fetch Response instance eventually provided to respondWith need not have the same URL as the request.  Because of the 1 channel = 1 URI invariant, an internal redirect must be performed in that case.  More on that later.

At a high level, redirects are simple:

    Create a new channel, propagating state from the original channel to the new channel.
    Check that the redirect is okay.  This is an opportunity run all the checks that were run when the channel was originally created.  Just because something is a redirect doesn't mean it's okay to violate the Content Security Policy or Mixed Content policies.
    If all the checks passed, notify the original channel so it can open the new channel, passing the original channel's listener so it now becomes the new channel's listener.  The listener will not have had its nsIRequestObserver::OnStartRequest method called yet because when a redirect response is observed, the redirect is performed rather than advancing to nsHttpChannel::ProcessNormal().

In practice, they're a bit more complex.  Here's a high-level sequence diagram of the (in the parent) nsHttpChannel redirection life-cycle using the StartRedirectChannelToURI programmatic redirect API used by nsIHttpChannel::redirectTo and the InterceptedChannel mechanism (more on that later).  For 3xx redirect responses, nsHttpChannel::AsyncProcessRedirection uses ContinueProcessRedirectionAfterFallback whose behavior is analogous to StartRedirectChannelToURI and whose ContinueProcessRedirection is analogous to ContinueAsyncRedirectChannelToURI/OpenRedirectChannel.

Sequence diagram of redirection without e10s details.

Elaborations on the diagram:

The nsAsyncRedirectVerifyHelper is also where fetch's [redirect mode](https://fetch.spec.whatwg.org/#concept-request-redirect-mode) is factored in.  If the mode is "follow", the redirect will be approved and proceed, as covered by the diagram.  However, if the mode was "error" (generate a network error on redirect) or "manual" (return the redirect response as an opaque-filtered redirect response) and the redirect wasn't internal or a Strict-Transport-Security upgrade, then the redirect will be aborted and the redirect itself will be the channel's result returned to the listener.

    When the original nsHttpChannel "Pumps [are] suspended", that's its nsInputStreamPump that provides the channel with OnStartRequest/OnDataAvailable/OnStopRequest events.  By suspending the pump, the channel doesn't need to worry about those calls happening.
    When checking if redirects are okay, nsIChannelEventSink::AsyncOnChannelRedirect methods are invoked.  This is an asynchronous check; the method needs to call the passed in callback to complete the check.  The nsAsyncRedirectVerifyHelper provides a DelegateOnChannelRedirect helper that abstracts the book-keeping for callbacks in addition to being the home to logic like invoking the nsIOService checks.
    The "nsIOService checks" AsyncOnChannelRedirect handler:
        Checks if this looks like a captive portal redirect to a local/private IP sub-net and hints to the nsICaptivePortalService that it should run a check again.
        Directly asks nsContentSecurityManager to vote on the redirect via its nsIChannelEventSink::AsyncOnChannelRedirect implementation.
        Asks every nsIChannelEventSink listed in the category manager under "net-channel-event-sinks" to vote on the redirect.  Currently that's the CSPService and the nsMixedContentBlocker.
    The listener, if it implements nsIChannelEventSink, is made aware of the redirect via a call to its AsyncOnChannelRedirect method with the result of all the prior checks.  It can veto the redirect or allow it to proceed.  If it proceeds, it can expect to receive an nsIRedirectResultListener::OnRedirectResult when the redirect has succeeded by AsyncOpen2-ing the redirected channel, or an error otherwise.

Sources of Redirects

When trying to understand code, I always find it useful to know the motivating use-cases.  Especially because these use-cases may interact in ways that increase complexity.  So, what causes redirects?

    Redirects initiated by nsHttpChannel as a result of content directives:
        Explicit 3xx responses from the network or cache.
        HTTP-to-HTTPS upgrades triggered by "upgrade-insecure-requests" Content Security Policy (CSP) directives or HTTP Strict-Transport-Security (HSTS).  A "permanent" "STS" redirect is performed.
    Redirects initiated by nsHttpChannel as a result of things going wrong:
        Failure to load a URL from the network that's covered by an Application Cache cache manifest FALLBACK entry that specific a file to load instead.  Note Application Cache (AKA "AppCache") has been deprecated in favor of Service Workers and is hopefully going away.
        Cache problems.  If we think we have a valid cache entry but it turns out we couldn't read the cache file from disk, then the cache entry is removed ("doomed") and we generate an internal redirect to hit the network.  This may also happen if certain edge-cases trigger when processing HTTP 304 "Not Modified" responses to conditional GETs.
        Proxy-server problems.  If the proxy server is unable to handle the request, an internal redirect will be generated to fail-over to the alternate proxy returned by nsIProtocolProxyService::getFailoverForProxy().
    Non-nsHttpChannel code calling nsIHttpChannel::redirectTo():
        WebExtensions using the webRequest API inspect/intercept HTTP requests that choose to perform redirects in onBeforeRequest or onHeadersReceived.  (If you are wondering if there's a way you can create an extension that does the same thing as a ServiceWorker for a site you do not control, this is the API for you.)
        Legacy Firefox Extensions.
        RecvRedirect2Verify **???**
**revised**
* ServiceWorker fetch events resolved with a Response whose URI does not match that of the request.  This is a special case that can't happen for normal web content because HTTP does not provide this capability.  The Location header only has meaning for redirect status codes and will
be ignored otherwise.
  * Pre-patch this happens in both the parent and child.  Post-patch this happens only in the parent.
* Service Worker-triggered Redirects via InterceptedChannelChrome using the internal nsHttpChannel::StartRedirectChannelToURI API call:
        A response was provided to respondWith() but its URI was different from the URI requested in the fetch event.  **up**
An internal redirect is generated so that a new channel can be created with the new URI as the one-channel-one-URI invariant demands.  This differs from a normal redirect where the ServiceWorker might be consulted again with a new "fetch" event generated and potentially respondWith()ed.

*
**DO EXPLAIN DO.**
    HandleAsyncAPIRedirect: Permanent.  Seems to be downstream of redirectTo.
        nsHttpChannel::BeginConnect checks mAPIRedirectToURI and uses AsyncCall to reach this. (6098)
    mAPIRedirectToURI: 2 side-effect cases.  One in ContinueProcessResponse1.  One in OnStartRequest.  One direct invocation in HandleAsyncAPIRedirect above.

ChannelEventQueue, a Nested Event Loop e10s IPC bandage

The normal event loop assumption is that events will run in the order they were enqueued, each event running to completion before the next event starts running.  Nested event loops like those used by synchronous XMLHttpRequest and synchronous/rpc Inter-Process Communication (IPC) calls violate this assumption.

The canonical Necko concern is that a call to an nsIRequestObserver::OnStartRequest implementation will be on the stack spinning a nested event loop, and that a call to the same listener's (nsIRequestObserver subclass) nsIStreamListener::OnDataAvailable implementation will be made.  This is not a problem for non-e10s channels because Necko event delivery has built-in back-pressure.  Core implementations like nsInputStreamPump and NS_AsyncCopy's nsAStreamCopier go out of their way to maintain thread-safe state machines that only allow events to be delivered after the previous event has run to completion.  Other low-level interfaces encourage idioms like this, for example nsIAsyncInputStream's asyncWait requests a single callback notification rather than a subscription to all future notifications.

Messages/events relayed via IPC can't have this free back-pressure.  At least as long as they're not "sync" IPC calls.  But, for reasons of performance and sanity, it is desirable for all IPC messages to be "async".  Additionally, there are distributed computing issues to deal with where both the parent and child may have different opinions of the channel's state and in-flight multi-step asynchronous operations.  Thus we have the ChannelEventQueue, a suspendable queue of ChannelEvent instances (basically non-XPCOM nsIRunnables).  When receiving IPC events, Necko *Parent and *Child classes will construct ChannelEvent subclasses and use ChannelEventQueue::RunOrEnqueue to ensure events are run sequentially.
E10s Complication: Diversions

Logic in the child content process may realize that the data it's receiving should be consumed by the parent process instead.  In that case the channel is diverted back to the parent process using nsIDivertableChannel.  It's currently used by:

    nsIExternalHelperAppService which handles downloads, asking you whether you want to "Open with" an external application or "Save File" as a download.
    PSMContentListener which handles importing certificates identified by the MIME types application/x-x509-*-cert.

These are both nsIURIContentListener implementations that are invoked by the nsIURILoader.  When a URL is opened (in a navigation context), the URI loader opens the channel to discover the content type.  This is initiated by a docshell in the (child) content process.  And since the content type comes from the "Content-Type" HTTP header, it means data has already started flowing to the child process.  Normally this is what we'd want because for HTML documents we want to parse and render them in the child.  But content processes are sandboxed and cannot directly write to disk, open applications, or manipulate the certificate store.

Sequence diagram of channel diversion, an inherently e10s activity.

The above sequence diagram expresses the high level parent-child diversion flow using the external helper app case as an example.  In prose:

    Not in the diagram: The child channel generates an OnStartRequest event, processed by nsURILoader's nsDocumentOpenInfo::OnStartRequest.  It invokes nsExternalHelperAppService::DoContent which defers to the content-process specific DoContentContentProcessHelper which creates an ExternalHelperAppChild and returns it.  nsDocumentInfo::OnStartRequest invokes ExternalHelperAppChild::OnStartRequest.
    The child channel consumer, ExternalHelperAppChild in this example, decides that it wants to hand processing off to the parent.  It invokes DivertToParent on the HttpChannelChild.
    HttpChannelChild suspends its ChannelEventQueue, pausing processing of data messages coming from the parent.  Because all of this is happening during the "OnStartRequest" handler, none of the "OnDataAvailable" events with the HTTP request's body have been processed.  HttpChannelChild then tells the parent about the diversion by creating a PChannelDiverter actor instance that can be used to claim the diverted channel in the parent process.
    When the parent receives the constructor message, it stops the flow of data by suspending the nsInputStreamPump that is delivering the OnDataAvailable notifications to the nsHttpChannel.  Prior to the diversion, these calls would be propagated to the HttpChannelParentListener and then onto the HttpChannelParent which would send the data to the child via OnTransportAndData messages.
    ExternalHelperAppChild conveys the PChannelDiverter actor to the parent in an IPC call to DivertToParentUsing(the diverter).
    ExternalHelperAppParent receives the IPC messages and invokes DivertTo(itself as stream listener).  This results in the HttpChannelParentListener switching its mNextListener from the HttpChannelParent instance to the ExternalHelperAppParent so that when data starts flowing, it will be the ExternalHelperAppParent receiving the OnDataAvailable notifications.  A synthetic OnStartRequest is also generated since the real OnStartRequest was already consumed by the child.
    HttpChannelParent's StartDiversion method sends two IPC messages to HttpChannelChild.  The first, FlushedForDiversion, gets wrapped into a ChannelEvent by the child and placed in the suspended ChannelEventQueue.  It will serve as a notification to the parent that the child has finished processing its event queue.  The second message, DivertMessages, resumes the processing of the ChannelEventQueue.  This will result in the child processing all of the buffered data messages.  Because diversion is enabled, they will be re-transmitted to the parent as DivertOnDataAvailable messages.  Finally, the FlushedForDiversion event will be processed, sending a DivertComplete message to the parent.
    HttpChannelParent processes the messages, and resumes the underlying nsHttpChannel, which will then resume its nsInputStreamPump, causing data and events to flow again.  The HttpChannelParentListener is still the nsHttpChannel's listener, but it will now be calling the ExternalHelperAppParent's OnDataAvailable.

E10s Complications: Redirects

Redirects get more complicated under e10s.  The good news is that "normal" redirects all happen in the parent, based around the same nsHttpChannel control flow discussed above.  The e10s logic builds on that.

foo DISCUSS foo
**TODO: e10s redirects**

**END: e10s redirects**

Service Worker and HTTP Channel Interactions
nsINetworkInterceptController

nsINetworkInterceptController is the high level interaction point between Service Workers and Necko.  Each "document" (really, nsDocShell) implements the interface.  This means it has the context to know whether the document is controlled and by whom in addition to any arguments passed in method calls.  The interface is exposed to the channel by being set as the notificationCallbacks attribute on the channel or the load group.

The interface defines 2 methods:

    bool shouldPrepareForIntercept(in nsIURI aURI, in bool aIsNonSubresourceRequest): A synchronous method to check whether the given URI should be intercepted and therefore should avoid touching the network.  If true is returned and the channel doesn't get canceled in the interim, a call to the next method should be expected at some point in the future.
    void channelIntercepted(in nsIInterceptedChannel aChannel): Here's an intercepted channel that you need to do 1 of 2 things to:
        Call resetInterception(), causing an internal redirect so that we act like the SW never existed, checking the HTTP cache and going to the network as appropriate.
        Generate a synthesized response.  Set the status via synthesizeStatus, set headers via synthesizeHeader, write to the responseBody stream,  and finish by invoking finishSynthesizedResponse(finalURLSpec).

Under e10s, the nsDocShell instances live in the content process.  So what happens in the parent process?  The answer is that HttpChannelParentListener also implements the interface and handles the calls.  nsHttpChannel always attempts to retrieve an nsINetworkInterceptController from its notificationCallbacks attribute and invoke its shouldPrepareForIntercept method to determine whether interception is appropriate.  This happens regardless of whether it's an e10s scenario where the nsHttpChannel was created by an HttpChannelParent or it's non-e10s and the nsHttpChannel was directly created in the parent.  nsHttpChannel doesn't distinguish between the two, it just consumes the interface.

### Intercepted Channels ###

**do DISCUSS MESSAGE FLOW AND CHANNEL AS NORMALIZING HANDLES do**.  Verify that channelcontent always bounces things of HttpChannelChild.

As mentioned above, nsIInterceptedChannel is the interface that the Service Worker fetch event handler uses to provide its Response object.  Currently this entails a number of separate method calls, but in the future it's likely the Response will be directly provided instead.

[DIAGRAM: intercepted-channels.svg]

There are two implementations of nsIInterceptedChannel: InterceptedChannelChrome and InterceptedChannelContent, both of which subclass InterceptedChannelBase.  Although the Chrome variant will only ever be created in the parent process and the Content variant in a child content process, they do not have a parent/child relationship like HttpChannelParent and HttpChannelChild.  We have two implementation types because they each hold mChannel references to the concrete channel types, nsHttpChannel in the parent, and HttpChannelChild in the child.  They do this because of their differing means of injecting the synthesized content and resetting interception.

In nsHttpChannel, Service Worker interception is performed during the OpenCacheEntry stage of processing.  If we intercept and provide a synthesized result, we never hit the HTTP cache and a synthesized cache entry is generated.  InterceptedChannelChome has the specific logic to populate the synthetic cache entry.  And it handles calls to resetInterception by manually triggering a programmatic redirect; nsHttpChannel itself has no resetIndirection method.

InterceptedChannelContent's behavior changes with the patch, which we'll get to in a later section.  The key difference to be aware of is that HttpChannelChild is very aware of ServiceWorkers and interception and so InterceptedChannelContent is able to be a thinner layer.  HttpChannelChild exposes a ResetInterception method it is able to invoke directly, and post-patch, content synthesis is simpler too.

Differences in behavior between e10s and non-e10s stem from whether the nsINetworkInterceptController is an nsDocShell (non-e10s in parent, e10s in child) or an HttpChannelParentListener (e10s in parent only).


**The Old Way: Child Intercept**
## Strategy: Don't Get The Parents Involved ##

The current pre-patch implementation optimizes for the ability to process Service Worker interceptions in the child with as little parent involvement as possible.  In a world where the decision to intercept and the processing of the intercept both happen entirely in the child process, this makes a lot of sense.  But it also comes with a massive amount of complexity because it means the child channel needs to duplicate logic that would normally be handled in the parent, plus the additional permutations when the parent does need to get involved.

The parent needs to be involved when any of the following things happen:
* The fetch needs to go to the network because neither respondWith() nor preventDefault() was invoked on the fetch event.
* respondWith() is resolved with a Response with a redirect status.
* Diversion: As previously covered, diversion is a mechanism by which the channel can be consumed in the parent process and by definition this involves the parent process.

The child can handle things without involving the parent process when:
* respondWith is resolved with a Response whose URI matches that of the request.  This is the simplest and most straightforward case.
* respondWith is resolved with a Response that does not have a redirect status but whose URI does not match that of the request.  This results in HttpChannelChild's specialized BeginNonIPCRedirect method being invoked to run the redirect entirely in the client.

### Complexity: Redirects ###

Redirects happen in the parent.  Why?  Because that's where they happened prior to the introduction of Serivce Workers.  Why?  Because nsHttpChannel already knew how to perform redirects and there's only a performance hit to remote the decision-making process from the parent where the network I/O is happening down to the child and then back up again.  (At least that's my guess on the simplest evolutionary explanation.)Redirect3Complete

As a result, if a ServiceWorker responds to a fetch event with a redirection, an HttpChannelParent will need to be spun up.  Things get more complicated because this transfers control to the parent and that redirected channel may also need to be intercepted.

As mentioned previously, HttpChannelParentListener implements nsINetworkInterceptController.  This is how the child regains control of the channel.  When responding with a synthesized redirect, the child tells the parent that it should intercept the redirected channel.  Its shouldPrepareForIntercept will then return true so that its channelIntercepted method will be called.  When it's channelIntercepted method is called it simply saves off the intercepted channel and does nothing.  Instead, the next action will be taken by the HttpChannelChild whose CompleteRedirectSetup method will be sending a SendFinishInterceptedRedirect ping-pong.  The parent will stop sending IPC messages on receipt and issue its own send.  The child will then delete the parent and continue the redirect handling in the child by invoking AsyncOpen2.
**need to document relationship of the self-deading channel to the redirected channel**.  It looks like the same child.  How do replacement channels work for this?

#### Normal Redirects Still Happen in the Parent ####

#### Known Synthesized Redirects Happen in the Child ####

### Providing the Synthesized Response ###

#### InterceptStreamListener ####

The synthesized response data and its nsIStreamListener events are actually being generated by HttpChannelChild's mSynthesizedResponsePump nsInputStreamPump.  This means that in the OnDataAvailable and other listener notifications, the nsIRequest* aRequest is that of the nsInputStreamPump rather than the HttpChannelChild.  This is potentially confusing to the listener.  So the InterceptStreamListener is registered as the pump's listener and it redirects each event to the child's listener, passing the child as aRequest instead.


**The New Way: Parent Intercept**

As stated previously, we are moving both the decision to intercept navigations and the processing of the intercept into the parent process

### Providing the Synthesized Response ###
