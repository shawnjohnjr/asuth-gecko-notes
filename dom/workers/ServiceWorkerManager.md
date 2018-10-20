# ServiceWorkerManager

## Overview ##



## Methods ##

### PUBLIC

* IsAvailable: Used by nsDocShell::ShouldPrepareForIntercept of
  nsINetworkInterceptController for non-subresource (AKA navigate).
* IsControlled: Used by nsDocShell::ShouldPrepareForIntercept of
  nsINetworkInterceptController for subresource (AKA non-navigate).
* DispatchFetchEvent: Used by nsDocShell::ChannelIntercepted

* Update: Called by SWR::UpdateInternal which is called by (mainthread)
  SWR.cpp:UpdateRunnable::Run
* SoftUpdate: Self-invoked by UpdateTimerFired, externally by
  SWMC::RecvNotifySoftUpdate
* PropagateSoftUpdate(2): self w/actor delay, plus service loopback.
* PropagateRemove: presumed same.
* Remove(host): Iterates over all mRegistrationInfos and their UserData,
  self-invoking ForceUnregister if HasRootDomain() matches  Self-invoked by
  RemoveAndPropagate and by actor-deferred runnable, externally by child in
  RecvNotifyRemove.
* PropagateRemoveAll
* RemoveAll

* GetRegistration(public: Principal, scope): variant of nsISWM getRegistration
  (and private one) this one used by:
  * ServiceWorkerClients' OpenWindowRunnable
  * ServiceWorkerUpdateJob::Unregister
  * ServiceWorkerPrivate::SendFetchEvent
* CreateNewRegistration
* RemoveRegistration
* StoreRegistration

* FinishFetch

* ReportToAllClients
* LocalizeAndReportToAllClients

* HandleError: Internally: used by ContinueDispatchFetchEventRunnable.

ServiceWorkerClients helpers
* GetClient: ServiceWorkerClients helper
* GetAllClients: ServiceWorkerClients helper
* MaybeClaimClient: (used by ClaimClients, thereby ServiceWorkerClients)
* ClaimClients: ServiceWorkerClients helper

ServiceWorkerGlobalScope helpers:
* SetSkipWaitingFlag: ServiceWorkerGlobalScope::SkipWaiting propagates to after
  thread jump.

* static GetInstance

Internal, but with moot-ish SericeWorkerManagerChild interactions:
* LoadRegistration:
  * SWMChild pokes in registration received from RecvNotifyRegister
  * LoadRegistrations uses recursively.
* LoadRegistrations
  * SWM::Init passes in after pulling out of SWR::GetRegistrations

* ForceUnregister: explicitly commented for use only by remove/removeAll

nsIServiceWorkerManager interface
* AddRegistrationEventListener
* RemoveRegistrationEventListener

* MaybeCheckNavigationUpdate

* SendPushEvent

* NotifyUnregister

* WorkIsIdle: recently added internalish check to determine whether the
  unregister logic can actually clear the registration or not yet; needs to wait
  for all extendable events to complete.

#### nsIServiceWorkerManager
ex
* register: Used by SWC::register
* unregister: Used by SWRWorkerThread::Unregister (which uses
  StartUnregisterRunnable to proxy to main thread).

ServiceWorkerContainer heleprs:
* getRegistrations(win): SWC helper
* getRegistration(win): SWC helper
* getReadyPromise(win): SWC helper
* removeReadyPromise(win): SWC helper

* getRegistrationByPrincipal: [TO:ISERVICE] used by devtools actor and test
  test_serviceworkerinfo.xul

* [notxpcom] MaybeStartControlling: nsDocument::SetScriptGlobalObject calls
  when being given a global (and not previously having one),  can be used for
  initial control plus also bfcache restore.
* [notxpcom] MaybeStopControlling:: nsDocument::SetScriptGlobalObject calls when
  the global is being cleared, such as normal closure or bfcache stashing.

SWR stuff that could/should be torn off.
* [noscript] GetInstalling: SWR helper only
* [noscript] GetWaiting: SWR helper only
* [noscript] GetActive: SWR helper only

* [noscript] GetDocumentController

* removeAndPropagate: [TO:SERVICE]
* getScopeForUrl: testing

* getAllRegistrations: For about:serviceworkers, maybe about:Debugging too.
  [TO:SERVICE]
* [implicit_jscontext] propagateSoftUpdate: [TO:MOOT] multiprocess workaround
  for the benefit of about:serviceworkers
* propagateUnregister: [TO:MOOT] same, about:serviceworkers hack

event sending
* sendNotificationClickEvent(...): [TO:SERVICE]
* sendNotificationCloseEvent(...): [TO:SERVICE]
* sendPushEvent: [TO:SERVICE]
* sendPushSubscriptionChangeEvent: [TO:SERVICE]

listeners:
* addListener
* removeListener

* shouldReportToWindow: ???



### PRIVATE

* Init
* GetOrCreateJobQueue
* MaybeRemoveRegistrationInfo
* GetRegistration(private: ScopeKey, Scope):
* AbortCurrentUpgrade
* Update
* GetDocumentRegistration
* GetServiceWorkerForScope
* GetActiveWorkerInfoForScope
* GetActiveWorkerInfoForDocument
* InvalidateServiceWorkerRegistrationWorker
* NotifyServiceWorkerRegistrationRemoved
* StartControllingADocument
* StopControllingADocument
* GetServiceWorkerRegistrationInfo(nsPIDOMWindowInner): helper, extracts
  document then calls (document) variant which chains all the way down.
* GetServiceWorkerRegistrationInfo(nsIDocument): helper, extracts principal and
  document uri from doc then calls (principal, uri) variant which chains.
* GetServiceWorkerRegistrationInfo(principal, uri): helper, converts principal
  to scope then uses (scopekey, uri) variant.
* GetServiceWorkerRegistrationInfo(scopekey, uri): ACTUAL work doer.
  * Uses (static) FindScopeForPath on spec to find CString scope, and
    RegistrationDataPerPrincipal 'data'.
  * Gets ServiceWorkerRegistrationInfo
* PrincipalToScopeKey
* AddScopeAndRegistration
* FindScopeForPath: Checks nsClassHashtable<cstring principal,
  RegistrationDataPerPrincipal> mRegistrationInfos.
* HasScope
* RemoveScopeAndRegistration
* QueueFireEventOnSreviceWorkerRegistrations
* FireUpdateFoundOnServiceWorkerRegistrations
* FireControllerChange
* StorePendingReadyPromise
* CheckPendingReadyPromises
* CheckReadyPromise

* AppendPendingOperation: used to track deferred actor-using ops when the actor
  does not yet exist.
* HasBackgroundActor: returns !!mActor

* MaybeRemoveRegistration
* RemoveAllRegistrations(OriginAttributesPattern): Iterates over all
  mRegistrationInfos' UserData()'s, invoking ForceUnregister for each match.
* NotifyListenersOnRegister
* NotifyListenersOnUnregister

tracking for error logging, partially
* AddRegisteringDocument:
* AddNavigationInterception: [TO:SERVICE] Tracks the nsIInterceptedChannel in
  the per-scope mNavigationInterceptions InterceptionList, adding an
  InterceptionReleaseHandle as the channel's release handle to automatically
  invoke RemoveNavigationInterception when the channel gets released.
* RemoveNavigationInterception: [TO:SERVICE] Called by the
  InterceptionReleaseHandle handled added by AddNavigationInterception.

* ScheduleUpdateTimer
* UpdateTimerFired
* MaybeSendUnregister

* SendNotificationEvent

## Methods in detail

### ::Register ###

* creates a `cleanedScope`
* AddRegisteringDocument tracks the [doc, scope] association for console
  reporting of failures.
* Creates a new `loadGroup` differing from `docLoadGroup`
* Creates a ServiceWorkerRegisterJob (isa ServiceWorkerUpdateJob) which has the
  cleanedScope on it.

## Jobs ##

* ServiceWorkerJob
  * ServiceWorkerUpdateJob
    * ServiceWorkerRegisterJob

### ServiceWorkerUpdateJob ###

# IPC / Parent/Child #

## Lifecycle ##
* On the main thread.
* ServiceWorkerManager::GetInstance news ServiceWorkerManager into gInstance.
* ServiceWorkerManager() constructor calls
  BackgroundChild::GetOrCreateForCurrentThread (just like RuntimeService::Init),
  which calls:
* SWM::ActorCreated:
  * constructs a (P)ServiceWorkerManagerChild
  * Dispatches any accumulated mPendingOperations.  (Accumulated inside
    AppendPendingOperation whose callers are all predicated on !mActor.)
    * Remove
    * PropagateRemove
    * PropagateRemoveAll
    * PropagateSoftUpdate
    * PropagateUnregister


## Propagate* ##
