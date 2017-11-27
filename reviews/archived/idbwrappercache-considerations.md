
I think there's 2 things going on:
1) IDB classes have always had a somewhat sketchy handling of shutdown and related GC scenarios, likely stemming from IDB being one of the first serious Worker-exposed APIs involving IPC and pre-dating bkelly's various WorkerHolder/related cleanups.  For example, IDBTransaction::WorkerHolder::Notify
2) Assertions firing.  Bug

Here's what I understand to be happening.  This WorkerHolder guaranteed

I don't think so. tl;dr: AFAICT (as far as I can tell) the cycle collection here exists to break the potential cycle between the

From code reading, mozilla::HoldJSObjects(this) as called by IDBWrapperCache::SetScriptOwner(owner)[1] does two things:
- Sets mScriptOwner, a JS::Heap<JSObject*> strong reference to the JS global (belonging to the window or worker).  This strong reference is dropped by the cycle collection unlinking process or the IDBWrapperCache destructor.
- Invokes mozilla::HoldJSObjects(this) which causes a weak reference to the IDBWrapperCache instance to be tracked by CycleCollectedJSRuntime in its mJSHolders array.  The weak reference is safe because IDBWrapperCache's destructor correspondingly invokes mozilla::DropJSObjects(this).



1: https://searchfox.org/mozilla-central/source/dom/indexedDB/IDBWrapperCache.cpp#60
