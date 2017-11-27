My searchfox todo's

## Comment/Megamenu Patch

* Handle narrow monitor behavior:
  * Force 2-line rows for the "Go to definition of SYMBOLDESC SYMBOLNAME" to
    attempt to address wide-ness.  Check out wrapping options.  Maybe grids?
    * Maybe try and go with viewport selector to accomplish this so we can avoid
      a resize hook.  Having said that, we're absolutely forcing a reflow
      anyways, so maybe neither are needed.  (We'd check body's clientWidth
      before injecting the menu and forcing its size to be computed.)
  * Perform positioning fixup so that:
    * horizontally, the top/primary menu is shifted as far left as possible so
      that the mouse is still vertically over the menu with some padding.  Maybe
      adjust the width of the secondary menu area if it would still exceed the
      screen width.
    * vertically, eh, punt for now.  the user can scroll.

 * Add minor "heuristics" to avoid comment foolishness:
  * NS_AsyncCopy is picking up a "//--------" delimiter comment on
    definition/declaration.

* Menu aim clearly needs to be tweaked; maybe a setting is wrong for

* Too many def/decl handling:
  * HttpBaseChannel::SetNotificationCallbacks is overflowing because the
    symbol gets "unified" (misnomer) with nsIChannel's notificationCallbacks
    attribute.  This results in a total of 17 entries, 1 of which is the idl
    and 16 of which are subclass related.
    * Need to better understand what's happening on the identifiers here.  This
      is a case where there was no jump and I synthetically injected one, which
      may exacerbate the problem.
    * I propose implementing a hybrid tree-ification logic here.  I think it
      makes sense to keep the nsIChannel.idl entry at the top, but the rest
      can be tree-ified.  This can then be used for Search (which also gets
      overflowed, it reports 38 uses in this case.)

Right, so, shadowing overloads need to be broken out more, for example, for
http://localhost:8000/mozilla-central/source/dom/quota/StorageManager.cpp#233
we end up with the immediate declaration plus the overrides that we really want
to break out into their own thing.

dom/quota/StorageManager.cpp #105

MainThreadRun() override;

dom/workers/RuntimeService.cpp #603

virtual bool MainThreadRun() override;

dom/workers/WorkerRunnable.h #400

virtual bool MainThreadRun() = 0;

dom/xhr/XMLHttpRequestWorker.cpp #231

virtual bool MainThreadRun() override;

## definition issues to look into

Currently shipping searchfox and dev server, when clicking on either part of
"nsIChannel::LOAD_REPLACE" brings up the auto-generated header.

## examples to diagram or what not

* The HttpBaseChannel HttpAsyncAborter mess with AsyncAbort(),
  HandleAsyncAbort(), AsyncCall(), etc.
