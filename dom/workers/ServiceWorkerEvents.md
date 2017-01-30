## File Contents ##

### AsyncLog helper ###

Log console message via aInterceptedChannel's nsIConsoleReportCollector

### CancelChannelRunnable ###
Helper to invoke nsIInterceptedChannel::Cancel on the main thread and ensure all
releases also happen on the main thread.

### ExtendableEvent ###
Note: lots of this is inline in the header.

### FetchEvent ###
isa ExtendableEvent

#### FinishResponse (anon) ####

main-thread

### PushMessageData ###
(webidl binding for the data on the event)

### PushEvent ###
isa ExtendableEvent

### ExtendableMessageEvent (for postMessage) ###
isa ExtendableEvent
