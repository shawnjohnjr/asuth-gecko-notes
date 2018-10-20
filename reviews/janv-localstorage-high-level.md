## Meta-ish issues ##

- The pref checks could now use StaticPrefs probably.


## Lifecycle ##

### Startup ###

### Shutdown ###

## Privacy Modes ##

### Private Browsing ###


### Session-only mode in the face of e10s ###

## Data Clearing ##

### ForgetAboutSite.jsm ###
The old approach is for dom/storage/StorageObserver.cpp to listen for
"browser:purge-domain-data" to trigger ClearMatchingOrigin to trigger the
database clear in the parent and then uses Notify() to propagate the clearing
operation to the children.

## Version Changes ##

### Upgrade ###

### Downgrade ###

### Downgrade-then-upgrade ###

## Info Propagation ##

- nsGlobalWindowInner::EventListenerAdded explicitly adds Observer and
  EventListenerRemoved explicitly removes.
