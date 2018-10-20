## Relation to Session History ##
- Separate, but SessionHistory needs a way to hold onto a given SessionStore.

## Can we use storage partitions ##
- The SessionStorage is akin to a userContextId

## Lifecycle Questions ##
- When to reap.
  - Session History restore can be delayed, can't reap immediately.  Probably
    want SessionHistory to have either a push or pull interface that indicates
    what is the current set of live id's.
