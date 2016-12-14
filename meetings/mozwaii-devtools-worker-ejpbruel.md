* Main server / root actor always in parent process.
* Technically actors are a flat list, but conceptually there's a tree.
* Pause actor lasts as long as the pause; when unpause, everything it owns gets
  cleaned up.

* Idioms:
  * request/reply
  * notification. ex: from breakpoint

* Actors:
  * Thread actor provides pause/play/etc.; the stepping API
  * Source actor: set breakpoints
    * breakpoint actor
      * frame actor

In content process, servers don't have root actor.

Lots of issues related to the worker code assuming it only has one global
(/ compartment).  So there are hacks and workarounds.  Such as avoiding using
native promises (which end up using the wrong global), and instead using
polyfill promises.

Can never create a wrapper in the content global to the debugger global.

Lazily create worker debugger.

worker server is identical to main thread content debugger servers.



TODO: talk to jryans about bulk packets.
TODO: talk to nick fitzgerald about memory tools to just paranoia check it all
  happens over the wire protocol.
