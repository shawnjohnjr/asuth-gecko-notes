# Splitting SessionStorage off from LocalStorage #
https://bugzilla.mozilla.org/show_bug.cgi?id=1322316

## Specific Patches ##

### part 1 - SessionStorage ###

#### Overview ####

The Storage class is refactored into a base class. (Storage.h, Storage.cpp)
* Methods that were previously concrete are now pure virtual, although there are
  some inline helpers implemented in terms of the pure virtual methods like
  NamedGetter.
* The nsSupportsWeakReference class is dropped from here.

LocalStorage is introduced as a subclass of Storage (Storage.h)
* It picks nsSupportsWeakReference up.

SessionStorage is introduced as a subclass of Storage (SessionStorage.h)

SessionStorage gets its own standalone SessionStorageCache (SessionStorage.h)
* It has some

#### Minutae ####
Storage.h:
* The StorageType enum:
  * Its values have gained "e" prefixes
  * It still lives on LocalStorage, **presumably temporary**

StorageManager.h:
* All changes are due to StorageType being refactored to be under LocalStorage.

StorageCache.h:
* All changes are s/Storage/LocalStorage/ name changes.


### part 2 - Rename Storage.{cpp,h} to LocalStorage.{cpp,h} ###

Extract LocalStorage.{h,cpp} out of Storage.{h,cpp}

### part 3 - SessionStorageManager ###

DOMSessionStorageManager renamed SessionStorageManager, extracted out of
StorageManager.{h,cpp} to SessionStorageManager.{h,cpp}

StorageType migrated from LocalStorage to Storage.



### part 4 - Rename StorageManagerBase to LocalStorageManager ###

StorageManagerBase renamed to LocalStorageManager.
DOMLocalStorageManager, a StorageManagerBase subclass that existed similarly to
DOMSessionStorageManager
StorageManager.h renamed to LocalStorageManager.h


### part 5 - Shared broadcasting of changes ###

### part 6 - Implementation of IsForkOf() ###

### part 7 - SessionStorageManager must be a StorageObserverSink ###

### part 8 - Rename StorageCache to LocalStorageCache ###
