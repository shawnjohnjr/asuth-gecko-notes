https://bugzilla.mozilla.org/show_bug.cgi?id=1411908

### Part 1

ActorsParent.cpp
- Database::RequestClose():
  - Idempotent guard via new mRequestedClose.
  - Invokes SendRequestClose() if the actor is still alive.
- QuotaClient::ShutdownWorkThreads()



Notes I started writing to document the protocol:

I'd suggest maybe adding comments to the various things:

- PBackgroundIDBDatabase: This actor is created by the IDBDatabase in the child.
  The child actor will remain alive until the last reference to the IDBDatabase
  is dropped, at which point the child actor will send a DeleteMe message to the
  parent, which will trigger the parent to Send__delete__.
