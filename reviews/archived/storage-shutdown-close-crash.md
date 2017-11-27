
### History.cpp sketchiness ###

tl;dr: The weak pointers are probably safe since they are dispatched to the
connection's async thread.

Bare/weak pointers held by:
* Runnables:
  * They are:
    * InsertVisitedURIs (isa Runnable)
      * Constructed by static InsertVisitedURIs::Start().  All of these get
        their database connections frmo a call to GetDBConn() that's held in a
        weak reference on the stack.
        * GetDBConn() returns mDB->MainConn()
    * RemoveVisits (isa Runnable)
  * They sketchily:
    * Save off mDBConn in their constructors, then use them in a scoped
      mozStorageTransaction inside their Run() methods.  The transaction
      acquires a strong reference to the transaction.
