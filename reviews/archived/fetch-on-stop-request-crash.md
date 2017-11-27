Extra credit to that crash reporter who described in detail how they produced the crash, and extra credit to :philipp as well for picking out this most excellent crash report and citing its useful comment!


The root problem is that nsHttpChannel::ContinueHandleAsyncRedirect[1] does not save off the NS_FAILED(rv) `rv` to mStatus if !redirectsEnabled.  In the particular situation from the crash in comment 0 and reproduced in my test, we:

- Create a cached redirect in order to trigger the HandleAsyncRedirect path from nsHttpChannel::ReadFromCache.
  - This is the only control flow path in which it is used.  Redirects as perceived by ContinueProcessResponse2 invoke AsyncProcessRedirection (which triggers the actual "async" step of gHttpHandler->AsyncOnChannelRedirect which does the verification) and its success-handling ContinueProcessRedirectionAfterFallback with an error/continuation handler of `ContinueProcessResponse3`.
  - The choice to initiate the redirect from ReadFromCache primarily appears to be an optimization since the decision is made based on headers and without needing to read the body.
  - The existence of HandleAsyncRedirect and its "RedirectAsyncFunc" "ContinueHandleAsyncRedirect" that effectively wraps the redirect process is exclusively to handle the case that the redirect fails.  The ContinueProcessResponse2 path eventual ends up in ContinueProcessNormal
