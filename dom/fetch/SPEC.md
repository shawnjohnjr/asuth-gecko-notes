Spec notes, primarily intended as a combination of digest, rote learning through
re-statement, and outstanding questions.  There may also be some related noted
in the serviceworker-sim repo, although those were more about referrer policy.

### Filtered Responses (related: response type) ###

The internal response is the true response for trusted platform functionality
like image decoding.  The filtered response is what content sees which is the
internal response with pieces lopped off for privacy/origin-safety reasons or
just to reserve some headers for the exclusive use of the user-agent and network
proxies.

Although responses have a response type that can be "default" which implies no
filtering, that's a sentinel value.  The response type is driven either by the
request's response tainting (propagated in step 13.2 of Main fetch: "basic" =>
"basic", "cors" => "cors", "opaque" => "opaque"), errors (=> "error"), or
redirect mode (in the case of "manual", => "opaqueredirect").

### Redirects ###

The RequestInit dict takes a `RequestRedirect redirect` enum value which can be
"follow" (default), "error", or "manual" which describe what to do when the
response is a redirect.  "error" is as it sounds (Response type "error"), and
"follow" follows the redirect using the HTTP-redirect fetch algorithm
(propagating CORS).  "manual" is special and saves the redirect response itself
as an "opaqueredirect" reponse type.

??? Why would you want an opaqueredirect type?  Like, who are you handing it to?
Is there an oracle

## Minutae ##

### "default": its many uses ###

* Request
  * cache mode: Meaningful, standard use-if-fresh, revalidate-if-stale, hit
    network normally otherwise, save the result with standard HTTP cache.
* Response
  * type: Seems to be the sentinel default value.
