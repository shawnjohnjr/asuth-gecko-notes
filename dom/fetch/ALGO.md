## Algorith Flows ##

* 5.1 Main Fetch can invoke:
  * 5.2 Basic fetch: (own origin && !CORS) || (mode == "no-cors")
  * 5.3 HTTP fetch
* 5.2 Basic Fetch can invoke:
  * hardcoded cases
  * 5.3 HTTP fetch
* 5.3 HTTP fetch can invoke:
  * service worker intercept
  * 5.5 HTTP-network-or-cache fetch (if not intercepted or no result)
  * 5.4 HTTP-redirect fetch (if redirect status and "follow")
  * 5.3 HTTP fetch (if had to auth)
* 5.4 HTTP-redirect fetch can invoke:
  * 5.1 main fetch (for tainting purposes)
* 5.5 HTTP-network-or-cache fetch can invoke:
  * 5.6 HTTP-network fetch (if no response)
* 5.6 HTTP-network fetch is just used by other things, but will write stuff into
  the HTTP cache.

## 5.1 Main Fetch ##

1. `response = null;`
11. Complicated if-cascade (inverted here for clarity):
  * Basic fetch if:
    * We're talking to our own origin and !(CORS flag)
    * It's a data URL and we weren't redirected (HTTP-redirect fetch clears the
      "same-origin data-URL flag" that is set by default when the request is
      created.)
    * It's an about: URL.
    * This is actually the fetch spec being reused for HTML navigation
      (mode="navigate") or we're the websocket spec being altered by fetch
      (mode="websocket").
  * Network error if:
    * mode is "same-origin" (because we would have done the basic fetch in the
      prior block if we actually we same origin.)
  * Basic fetch, with response tainting set to "opaque" (which causes the
    response to be an opaque filtered response which is basically all empty/0)
    * mode is "no-cors"
  * Network error if:
    * Scheme is not http/https.  (We already handled data/about/websockets.)
  * HTTP fetch using { CORS, CORS-preflight }, with "cors" response tainting
    and "error" redirect mode, and clearing matching cache entries if there was
    an error (presumably for security/privacy reasons.) IF:
    * TODO
  * HTTP fetch with { CORS }, "cors" response tainting
(response processing)

## 5.2 Basic fetch ##

Dispatch based on scheme:
* about: hardcoded cases:
  * about:blank
  * about:unicorn
* blob:
  * pierce the url object to get the underlying blob
  * obvious content-length/content-type stuff
  * propagate https state of the client
  * obvious read/stream and return
* data:
  * Only works on GET
  * Synthesize content-type header from the data url
  * propagate HTTPS state from client
  * return that or network error otherwise
* file: underspec'ed
* ftp: underspec'ed
* filesystem: underspec'ed
* http, https:
  * perform HTTP fetch
* else:
  * network error


## 5.3 HTTP fetch ##

3. Perform service worker intercept unless we were told to skip it (or we are a
   service worker or there's no controlling service worker.)
4. If we skipped service worker or it had no response for us:
  1. Do a CORS preflight if requested and we don't have a cached preflight.
  2. Set the skip-service-worker flag so if redirects happen we don't ask the
     service worker again.
  3. ~credentials
  4. Perform HTTP-network-or-cache fetch, propagating credentials
     and authentication-fetch flag states.
  5. If CORS, do a CORS check and return network error if it didn't pass.
5. Branch on response:
  * redirect status:
    1. (HTTP/2 RST_STREAM if anything but a 303)
    2. Branch on redirect mode.
      * "error" => set response to a network error, will turn into a rejection
      * "manual" => save off the redirect in an opaqueredirect.
      * "follow" => yeah, redirect.  Run HTTP-redirect fetch propagating CORS.
  * 401 (Unauthorized): Return the response unless: this isn't CORS, we're
    associated with a window, and credentials mode is "include": in which case
    we should prompt the user and re-run HTTP fetch with the credentials (and
    authentication-fetch flag set.)
  * 407 (Proxy authentication required): Prompt for that and re-run HTTP
    fetch.
  * else: fall through
6. (something with authentication entry stuff for proxies)
7. Return response

## 5.4 HTTP-redirect fetch ##

1. Pierce any filtered response to get at the internal response as the actual
   response.  (Caller was responsible for security.)
2. through 6: Parse the Location header, interpreting it relative to the
   URL of what we were trying to fetch since HTTP/1.1 lets it be relative.
   Return a network error if any parse failed.
7. Network error if our redirect count hit twenty.
8. Increment redirect count.  (So postfix)
9. Clear the same-origin data-URL flag (which causes "follow" to explode if
   you give it a data URL.)
10. If request mode was "cors" and this redirect is cross-origin and it
    includes credentials, network error.
11. If the CORS flag is set and the redirect includes credentials, throw a
    network error.  (This happens if we were cross-origin but got redirected
    back to our own origin.)
12. If the CORS flag is set, the redirect is to a different origin from the
    request, then modify the request's origin to be a GUID.  (I think this is
    further protection against redirects back to the original origin.  Once
    you bounce out of your origin you should not be able to get bounced back
    into it.)
13. If this was a POST and we got a 301, 302, or 303, switch to GET and nuke
    the request body.
14. Append the new location to the request's url list.
15. Invoke main fetch propagating CORS and setting recursive.  (The note says it
    goes via main fetch for response tainting purposes.)

## 5.5 HTTP-network-or-cache fetch ##

1. Possibly clone the request so we can mess with it.
(3-14: build up the headers)
16. Check the cache for complete responses, possibly using it.
  1. If "force-cache"/"only-if-cached" use the response, don't do revalidation
     stuff.
  2. If "default" and the cache is fresh, use the response, and set the
     internal-only not-really-used-by-the-spec cache state to "local".
  3. Otherwise if "default"/"no-cache", modify the http request with
     revalidation headers (and don't set `response`).
17. Check the cache for partial responses, and if cache mode is "default" or
    "force-cache", modify the http request to have resume headers
18. Still no `response`?
  1. Bail if "only-if-cached", returning a network error that will become a
     rejection.
  2. Run *5.6 HTTP-network fetch* propagating the credentials flag, this can be
     either a from-scratch fetch or a revalidation or resume fetch.
19. Handle 304 not modified if "default"/"no-cache":
  1. Pull out the cached response.
  2. Explode with network error if there wasn't one.
  3. Update the cached response's headers from what the network told us, per
     HTTP spec.
  4. Now use that freshened cache response.
  5. Set cache state to "validated".
20. Return

## 5.6 HTTP-network fetch ##

2. Get a websocket or normal HTTP-ish connection, depending.
3. Network error if unable to get a connection.
4. Do the request to gobble the headers, possibly showing a TLS client cert
   dialog.
(5-9 various streaming/flow control setup)
10. Weird content-encoding stuff.
11. Propagate URL list to response from request.
12. Set CSP on response.
13. If it's not a network error and cache mode wasn't "no-store", update the
    HTTP cache.
14. Set cookies based on headers (unless the user agent says otherwise.)
15. Do the body stream stuff "in parallel"/asynchronously.
16. Return the response.
