### Bug 1228937 gFactoryOps->IsEmpty() assertion blocking bug 1246828 on shutdown phase ##

#### Crash stack

Basically looks like:
* Factory::ActorDestroy (inlined-ish? missing its own explicit frame)
* Receiving notification of the child having shut down?
  * mozilla::ipc::PBackgroundParent::DestroySubtree
  * mozilla::ipc::PBackgroundParent::OnChannelClose
  * mozilla::ipc::MessageChannel::NotifyChannelClosed
* (spun event loop)
* mozilla::dom::indexedDB::ConnectionPool::Shutdown [dom/indexeddb/ActorsParent.cpp:11686]
* mozilla::dom::indexedDB::QuotaClient::ShutdownWorkThreads [dom/indexeddb/ActorsParent.cpp:17158]
* mozilla::dom::quota::QuotaManager::Shutdown() [dom/quota/ActorsParent.cpp:3079]
* mozilla::dom::quota::QuotaManager::ShutdownRunnable::Run() [dom/quota/ActorsParent.cpp:2311]
* (PBackground spinning))

Factory::ActorDestroy is only doing that because sFactoryInstanceCount hit 0.
The unsurprising counterpart is Factory::Create.

Assertion sorta contradicts the code in FactoryOp::DirectoryOpen that checks for !gFactoryOps.

#### gFactoryOps stuff



* Places things get appended:
  * FactoryOp::DirectoryOpen()
    * called by FactoryOp::DirectoryLockAcquired which is the result of the
      QuotaManagerOpen stage.  That calls QuotaManagerOpen which calls
      OpenDirectory which invokes the same on the QuotaManager, with the
      OpenDirectorySignature netting us that callback signature.
    * sets state to DatabaseOpenPending and directly invokes DatabaseOpen
    * may also mark itself as delayed; walks from the end of the list backwards
      finding an op that it MustWaitFor, saving itself in the mDelayedOp.  This
      transitively creates a chain; delayed ops don't have DatabaseOpen invoked
      and instead must wait for the thing to complete and schedule its runnable.

* Removed:
  * FactoryOp::FinishSendResults()
    * Called by:
      * OpenDatabaseOp::SendResults
      * DeleteDatabaseOp::SendResults
    * Those are called by FactoryOp::Run when in State::SendingResults
