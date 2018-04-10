
== digression
I believe the issue :bkelly was trying to get at was that without an OriginAttributes-based partitioning mechanism, this is a backdoor for tracking scripts to reach further than the policy says they should.  Specifically, if we allow the iframed "the-game.tld" to load the Facebook tracking scripts and communicate with facebook.com, those scripts can then:
- Bi-directionally communicate in real-time with other "the-game.tld" windows via BroadcastChannel or the Clients API, even though those Windows themselves cannot directly load Facebook.com scripts or communicate with facebook.com.
- Interact with IndexedDB/localStorage/DOM cache API storage used by other "the-game.tld" windows.  This means:
  - "the-game.tld" can log tracking information to IndexedDB when it's top-level and not allowed to talk to Facebook.com, and send the data when it is loaded in an iframe.  (This would not be the case if partitioned using firstPartDomain.)
  - The facebook.com tracking scripts can be "laundered" so that they can be evaluated in the-game.tld's origin.

That said, I don't understand the threat model we're dealing with.  It seems like the-game.tld could just embed a facebook.com iframe on its site any time it wants to tell it tracking information to log.  It mainly becomes an issue of deployment logistics, since the-game.tld does need to communicate with the iframe via some glue code, and the-game.tld would need to host that themselves at some level.  If we'd block the iframe in such a case, then maybe the partitioning becomes a bigger issue, because it does open the way for interstitial landing/redirect pages that use IndexedDB as a dead-drop, assuming the tracking site is allowed to iframe embed the-game.tld even if the-game.tld isn't allowed to do that itself.
== /digression
