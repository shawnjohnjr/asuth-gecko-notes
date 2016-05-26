## What Lives Where ##

* Main Process
  * SpecialPowersObserver.jsm: Root JS logic, imported by enclosing bootstrap.js
    extension thing which also registers an XPCOM SpecialPowersObserver(Factory)  
  * SpecialPowersObserverAPI.js: Loaded by the SpecialPowersObserver.jsm using
    the subscript loader which subclasses SpecialPowersObserverAPI provided by
    this file

* specialpowers.js
* specialpowersAPI.js

## General Operation ##

### SpecialPowersObserver ###
(Lives in main process, loaded by extension's bootstrap.js).

* Registers with message manager for SP* messages.
* Uses loadFrameScript to cause (MozillaLogger.js,) "specialpowersAPI.js", and
  "specialpowers.js" to be loaded in *every* frame as a frame script in their
  own scope.
