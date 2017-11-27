
## Necessity ##
Meh

The initial LocalStorage "storage" e10s propagation in
https://bugzilla.mozilla.org/show_bug.cgi?id=1285898 did two particularly bad
things from a performance/efficiency perspective:
- All broadcasts go everywhere without filtering.
- Correctness changes limited our ability to precache things because the
  invariants for broadcast and storage were not the same.


## Diagnostic Support Stuff ##

```
(function() { var valueBytes=0; var keyBytes=0; for (var i=0; i < localStorage.length; i++) { var key = localStorage.key(i); keyBytes += key.length; valueBytes += localStorage[key].length; } console.log("localStorage has", localStorage.length, "keys and uses", keyBytes, "key bytes,", valueBytes, "value bytes,", keyBytes+valueBytes, "total bytes-ish"); })();
```
