https://bugzilla.mozilla.org/show_bug.cgi?id=1478602

quasi-abandoned notes because SQLite should be enforcing sanity here.

The most likely situation that jumps to mind is:
- Instance 1 of the extension starts the upgrade.
  - This results in the VersionChangeOp updating the "version" field of the database prior to actually starting the upgrade transaction in the child.  We update the version at https://searchfox.org/mozilla-central/rev/943a6cf31a96eb439db8f241ab4df25c14003bb8/dom/indexedDB/ActorsParent.cpp#22686 and only start the upgrade transaction after that, below.
- Instance 2 of the extension begins to load while instance 1 is still tearing down.
  - Instance 2's OpenDatabaseOp checks the database's version at https://searchfox.org/mozilla-central/rev/943a6cf31a96eb439db8f241ab4df25c14003bb8/dom/indexedDB/ActorsParent.cpp#21587 before getting to the point that it bothers to find out if there are other open connections.  This means it sees the new/upgraded version!  This is normally what we want because we don't want redundant upgrades stacking up.
  - Instance 2 decides that it doesn't need to perform an upgrade and goes straight to sending results.
  - So instance 2's open will succeed, although  

Thank you for providing a reproducible test case and doubly so for the video!  Here are my notes thus far, and I think I have a question for you...

## The error reported by the page in the video

As I understand what's happening in the repro case, the line:
  const getAllRequest = db.transaction(DB_IMAGE, DB_OP_READONLY).objectStore(DB_IMAGE).index(GROUP_INDEX).getAll(groupId);
in background_gsd.js is synchronously throwing an error in the async function getThumbnails, which is converted from an exception to a rejection, and that rejection is part of the promise-chain returned from the browser.runtime.onMessageg.addListener() handler added earlier in the file.  The MessageManager is doing its toString fallback coercion because the Error won't structured clone.  And it's that stringified error that's appearing in the window.

Given that IndexedDB updates its content-page structures synchronously, it seems very difficult for this to go wrong if the update ran to completion.  However, this does make absolute 

## The browser console error about updating the addon


This begs the question of whether the addon's upgrade mechanism is doing something to violate the atomic nature of the schema upgrade.  If the background page is being aggressively killed, it's possible this 
