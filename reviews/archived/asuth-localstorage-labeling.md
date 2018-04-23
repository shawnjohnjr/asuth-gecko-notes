## Semantics ##

### What does Chromium/Blink do? ###

#### Overview

There's a hybrid of an old and new system.  The old system appears to use SQLite
while the new implementation is mojo (their IPC layer) + (remoted-ish via mojo)
LevelDB.

Relevant links:
- LevelDB proxy magic:
  https://cs.chromium.org/chromium/src/components/leveldb/leveldb_mojo_proxy.h
- Browser process pieces (equivalent to Gecko chrome/parent process):
  https://cs.chromium.org/chromium/src/content/browser/dom_storage/
- Common pieces (browser and renderer):
  https://cs.chromium.org/chromium/src/content/common/dom_storage/
- Render
  https://cs.chromium.org/chromium/src/content/renderer/dom_storage/
#### Quota

Allows a 100k overage to deal with multiple renderer processes arriving at an
over-quota situation.
