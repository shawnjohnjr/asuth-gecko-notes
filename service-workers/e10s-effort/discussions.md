## September 20th: e10s meeting
nothing of note

## August 30th: e10s meeting
nothing of note.

## August 25th: bkelly

Ben wants:
* only ship Request/Response over the wire.

Terminology:
* lifecycle as statemachine

PServiceWorker: on PContent, ability to spin up a ServiceWorkerPrivate
independent on its own.
* it has SendFetchEvent

nsIInterceptedChannel as able to produce the Request
* synthesizeResponse as taking a Response.

Avoid necko's PHttpChannel entirely.  Potentially do away with
nsINetworkInterceptController.

Separate directory for ServiceWorkerPrivate.  Abstract interface/listener.
