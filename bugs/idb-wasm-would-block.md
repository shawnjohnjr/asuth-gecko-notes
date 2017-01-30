https://bugzilla.mozilla.org/show_bug.cgi?id=1319531

### Probably ignoreable quota manager errors about touching .metadata-v2

(This might be an artifact of how the databases are nuked by resetting the
quota manager?  Seems sketchy, honestly.)

The WARN_IF cascade looks like this:
[Parent 27287] WARNING: '!aOutputStream', file /home/visbrero/rev_control/hg/mozilla-central/xpcom/io/nsBinaryStream.cpp, line 125
[Parent 27287] WARNING: 'NS_FAILED(rv)', file /home/visbrero/rev_control/hg/mozilla-central/dom/quota/ActorsParent.cpp, line 1838
[Parent 27287] WARNING: 'NS_FAILED(rv)', file /home/visbrero/rev_control/hg/mozilla-central/dom/quota/ActorsParent.cpp, line 5700
[Parent 27287] WARNING: 'NS_FAILED(rv)', file /home/visbrero/rev_control/hg/mozilla-central/dom/quota/ActorsParent.cpp, line 5545
[Parent 27287] WARNING: 'NS_FAILED(rv)', file /home/visbrero/rev_control/hg/mozilla-central/dom/quota/ActorsParent.cpp, line 5409

Where the general context is an attempt to touch the metadata-v2 for the
origin mOrigin = "http://mochi.test:8888" with
mPath = "/tmp/tmpThfRU9.mozrunner/storage/default/http+++mochi.test+8888"

but where the profile only has, on-disk,
./permanent/chrome/idb/2918063365piupsah.files
./permanent/chrome/idb/2918063365piupsah.sqlite-wal
./permanent/chrome/idb/2918063365piupsah.sqlite
./permanent/chrome/idb/2918063365piupsah.sqlite-shm
./permanent/chrome/.metadata-v2
./permanent/chrome/.metadata

### -P wasm test-run

Figured out the errors, documented in comment 29.

with storage nuked, no session restore so fresh load of URL.

immediate crash/sadness!

.
./.metadata
./.metadata-v2
./idb
./idb/301792106ttes.sqlite
./idb/301792106ttes.sqlite-wal
./idb/301792106ttes.sqlite-shm
./idb/301792106ttes.files
./idb/301792106ttes.files/1
./idb/301792106ttes.files/journals

first weird error:
[Child 28541] WARNING: attempt to modify an immutable nsStandardURL: file /home/visbrero/rev_control/hg/mozilla-central/netwerk/base/nsStandardURL.cpp, line 1630

PID now 30322

first relevant seeming error:
[Parent 28489] WARNING: Failed to removed journal!: file /home/visbrero/rev_control/hg/mozilla-central/dom/indexedDB/ActorsParent.cpp, line 11872
