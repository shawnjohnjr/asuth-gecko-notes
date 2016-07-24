## PContent ##

* child:
  * Service Worker Infra
    * InitServiceWorkers
  * Push API:
    * Push/PushWithData
    * PushSubscriptionChange
    * PushError

### Push API ###

PushNotifier.h/PushNotifier.cpp has a sorta recursive relationship with
ContentParent.  PushMessageDispatcher::SendToChild takes a ContentParent
and invokes SendPush/SendPushWithData which in turn will create a
PushMessageDispatcher, albeit which should then take a different course of
action.  (In the parent, it wants to flood itself to all content processes.)

PushMessageDispatcher::NotifyWorkers gets the local ServiceWorkerManager and
invokes SendPushEvent.
* It seems like this should just be offloaded to ServiceWorkerManagerParent to
  route appropriately.

PushMessageDispatcher::NotifyObservers/DoNotifyObservers uses the category
manager on "push" to instantiate services to make sure they're around to hear
pushes.  The actual observer topics are "push-message",
"push-subscription-change", and "push-subscription-modified".  They are exposed
as strings on nsIPushService for a somewhat awkward contract.  (The main upside
is you can register weak observers?)

The only current consumer right now is the FxAccounts mechanism that lives at
services/fxaccounts/, specifically FxAccountsPush.js.  We can check that via:
* Searches on the category registration (required for ensured startup):
  https://dxr.mozilla.org/mozilla-central/search?q=%22category+push%22
* Searches on pushTopic (required to subscribe to the topic safely):
  https://dxr.mozilla.org/mozilla-central/search?q=pushTopic&redirect=false

This means that we can simplify the PushNotifier logic to just ensure that the
observer notifications happen in the parent, same as we'll change the worker
mechanism.

## PTabContext.ipdlh

* UnsafeIPCTabContext exists for ServiceWorkerClients::OpenWindow

## postMessage ##

Q: How are these routed around for content?
A: It seems like they aren't:
* SW -> Client/Page: (Right now workers can't be clients, tracked as
  https://bugzilla.mozilla.org/show_bug.cgi?id=1032521).
  ServiceWorkerClient::PostMessage creates a runnable that requires a window,
  populates a JS context and creates a ServiceWorkerMessageEvent that it
  directly dispatches on the window.
* In general, other postMessages work similarly.  The thing that will receive
  the message is directly accessible or otherwise exposes a handle object that
  exposes a postMessage implementation that knows how to create a runnable that
  will create a DOM event of some type that will be DispatchEvent'ed.

Q: How do remote mozbrowser frames handle this?  Is there just an explicit
   mozbrowser API that bridges the boundary?
A: Looks like they don't, not really.  It looks like most stuff would just be
   explicitly bridged using BrowserElement{Parent,Child} and the message
   manager on an as-needed basis.  And CPOW's presumably are used for any
   stop-gap.

* Better understand how MessagePorts work.  Clearly there's a transitive
  reachability graph at play that ostensibly would limit escaping the process.
  But there is a PMessagePort managed by PBackground.  Is it just a forward
  looking way to easily bounce between PBackground and back?  (But workers do
  much on the main thread, right?)
* Relatedly, check control flow path for BroadcastChannel.  Likely to be
  interesting... or

### Existing PostMessage exposure ###

Service Workers:
* To ServiceWorker:
  * SW::PostMessage uses SWP::SendMessageEvent.
    * SpawnWorkerIfNeeded()
    * (m)WorkerPrivate::PostMessageToServiceWorker
      * PostMessageInternal creates MessageEventRunnable which is a
        StructuredCloneHolder and normal Runnable, writes into it, and
        dispatches to the main thread.  (Which is currently an assumption about
        same-process document-onlyclients.)
* From ServiceWorker:
  * ServiceWorkerClient::PostMessage creates
    ServiceWorkerClientPostMessageRunnable which is a StructuredCloneHolder and
    WorkerRunnable, writes into it and dispatches it.
Shared Workers:  They already use a message port under the hood! Woo!!!
* To SharedWorker:
* From SharedWorker:


Shared Workers:

### Design Options and Trade-Offs ###

#### Proposal

still pending steps:
* IPDL wire protocol and metadata identifying client/origin.  For ServiceWorker,
  is this data currently baked into the runnable, or is it being encoded in the
  structured clone and I missed it?

#### Considered

Maintain existing types, internalize concept of remote existence:
* Good
  * Avoid need to add more vtable overhead.
* Bad
  * This can leave a lot of effectively meaningless functionality present that
    needs to be guarded.  Arguably better to just have a minimal interface
    class.

Use IPDL Parent/Child in uniform way somewhat like IndexedDB does:
* Ponderings:
  * IndexedDB always performs all meaningful activity in the parent in
    PBackground.  There is no need to communicate between siblings, but this is
    exactly what the remote worker scenario entails.

Attempt to reuse MessagePort infrastructure, similar to how SharedWorker reuses
it.
* Bad:
  * Awkward fit.


### Remote-Awareness Considerations ###


Service Workers:
* From ServiceWorker:
  * Clients inherently known.

### Possible Remote-Awareness Options ###



## SharedWorkers ##

SharedWorkers are not magic.  They are newed via their WebIDL binding which
invokes SharedWorker.cpp's static SharedWorker::Constructor.  It in turn uses
RuntimeService::CreateSharedWorker
