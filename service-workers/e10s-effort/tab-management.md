## Impl Hierarchy Relevance ##

? GeckoChildProcessHost

### ContentProcessManager ###
Parent process singleton responsible for book-keeping of:
* ContentParent instances keyed by ContentParentId
* Their known tabs (keyed by TabId) and their RemoteFrameInfo as value (opener
  TabId, plus TabContext).  Where TabParent inherits from TabContext, so this
  can be the TabParent.

### ContentParent ###
inherits from:
* PContentParent
* nsIContentParent (hand-rolled .h)

### TabParent ###
Inherits from:
* PBrowserParent
* nsITabParent
* TabContext

### TabChild ###

Inherits from:
* PBrowserChild
* nsITabChild: Seems like tagging interface which can be used to hang IPC e10s
  era things off of: its nsIContentFrameMessageManager and nsIWebBrowserChrome3
  (which holds e10s specific hack things), plus some hacky scriptable exposure
  of PBrowser stuff (ex: remoteSizeShellTo).
* TabContext
* nsIWebBrowserChrome*

Creates and holds nsIWebBrowser as nsIWebNavigation mWebNav and does all the
relevant top-level stuff like invoking nsIBaseWindow.Create, setting the
docshell type, etc.

## Life-cycle Stuffs ##

### Names ###

* ContentParentId: Names processes/ContentParent instnaces, allocated by
  ContentParent::InitializeMembers from gContentChildID++.
* TabId: Allocated from ++gTabId by ContentProcessManager::AllocateTabId.

Note that the ContentParent's ContentParentId is exposed on it as its
ChildID/mChildID in a slightly confusing bit of nomenclature.

### Process References ###

LinkedList<> in-object single-list thing of inherently ALL content parents:
* sContentParents

ContentParent* array/hashes that maintain subset lists.  MarkAsDead() knows how
to remove all of them from their lists.
* sAppContentParents: hashtable keyed by its mAppManifestURL.  Added to by
  CreateBrowserOrApp in the non-browser case.
* sNonAppContentParents: Array, added to via GetNewOrUsedBrowserProcess when it
  does PreallocatedProcessManager::Take or is forced to new a ContentParent.
  MarkAsDead handles cleanup.
* sPrivateContent: Array,

#### Choosing the ContentParent ####


#### Who says to create the process?

In the end, nsFrameLoader ends up calling ContentParent::CreateBrowserOrApp
which can cause a new process to come into existence via
ContentParent::GetNewOrUsedBrowserProcess.

We can generally end up in nsFrameLoader if:
* nsWindowWatcher::OpenWindowInternal gets invoked via OpenWindowJS or the C++
  OpenWindow.
  * (nsIWindowProvider::ProvideWindow is deferred to (or a new chrome window is
    created, which we don't care about for the purposes of this analysis.))
* Someone creates a mozbrowser iframe and nsFrameLoader::ShouldUseRemoteProcess
  returns true (saved to mRemoteFrame).  This depends on prefs, most likely
  "dom.ipc.browser_frames.oop_by_default" and whether remote=true was specified
  as an attribute of the iframe.  


nsIWindowProvider::ProvideWindow does!  Core control happens via
nsWindowWatcher::OpenWindowInternal.  Which is common helper of OpenWindowJS
and OpenWindow.  There's some neat special casing based on userContextId in here
(via CheckUserContextCompatibility).

ProvideWindow implementations, and when used:
* nsContentTreeOwner: If the `parent` existed (probably) and wasn't a chrome
  window, and it's not a dialog request.
  * Tries to open a sibling mozbrowser IN-PROCESS iframe via
    BrowserElementParent::OpenWindowInProcess.  The created iframe is explicitly
    not remote, so we do not care about this for process creation.  (See
    ContentChild for the sibling case of OpenWindowOOP.)    
* ContentChild: via nsContentUtils::GetWindowProviderForContentProcess() that
  exists just to expose ContentChild::GetSingleton() (hack), happens when in a
  content process and "we don't have a tabchild we can use" (which happens when
  there was no `parent` probably?  Unless it existed and lacked a docshell?)
  * If mozIframe: Locally, in-process, creates TabChild/PBrowser, then does
    SendBrowserFrameOpenWindow to trigger:
    * TabParent::RecvBrowserFrameOpenWindow calls
      BrowserElementParent::OpenWindowOOP.  Doc in PBrowser.
  * If not mozIframe: calls SendCreateWindow:
    * ContentParent::RecvCreateWindow.
      * Uses RAII AutoUseNewTab to inform call cascade that will bottom out in
        ContentParent::CreateBrowserOrApp. THEN
      * nsFrameLoader gets created, and it knows it's dealing with a remote.
      * nsFrameLoader::ReallyStartLoadingInternal calls TryRemoteBrowser() if
        IsRemoteFrame() && !mRemoteBrowser.  OR  nsFrameLoader::ShowRemoteFrame
        does it (called by Show).  AND no one has called SetRemoteBrowser with
        a browser.
        * The SetRemoteBrowser case happens in the
          BrowserElementParent::OpenWindowOOP case, in which case the provided
          aPopupTabParent is explicitly provided and used.
      * We bottom out in nsFrameLoader::TryRemoteBrowser which calls
        ContentParent::CreateBrowserOrApp.  (There's also
        nsFrameLoader::SwitchProcessAndLoadURI, but it seems to only have a
        single JS caller, test_frameLoader_switchProcess.html, which shows
        signs of being b2g related, so it might be doomed.)
* TabChild: ???
* else: window watcher does it itself and creates a new chrome window and it
  will pull the `newDocShellTreeItem` out via do_getInterface.

== Other sorta paths
*

== Intermediary Analysis
(comesfrom tree)
* Somebody possibly uses AutoUseNewTab
  * window.open(): ContentParent::RecvCreateWindow
    * ContentChild::ProvideWindowCommon() calls SendCreateWindow.
      * ContentChild::ProvideWindow
      * TabChild::ProvideWindow.  Notably capable of returning current window
        for OPEN_CURRENTWINDOW versus calling ProvideWindowCommon.
* (static) ContentParent::CreateBrowserOrApp().  Uses
  TabParent::GetNextTabParent() to get at window.open()'s stashed "hey use
  this window for touchability reasons" stuck there via the RAII AutoUseNewTab.
  It is sNextTabParent and has friends mCreatingWindow and mDelayedURL, all
  extensively documented in TabParent.h.

== Earlier analsysis

GetNewOrUsedBrowserProcess (static, returns refered ContentParent) is our
nominal entry point.  It gets called by:
* ContentParent::RecvCreateChildProcess
  * (static) ContentParent::CreateContentBridgeParent
    * (static) ContentParent::CreateBrowserOrApp, two spots:
      * When aContext .IsMozBrowserElement or .HasOwnApp().
      * When isInContentProcess

#### Launching ####

Processes get launched via ContentParent::LaunchSubprocess.  This happens from:

b2g-specific?:
* ContentParent::RunNuwaProcess

Normal path:
* ContentParent::GetNewOrUsedBrowserProcess
But which can use preallocted process mechanism that seems to be relevant if
somewhat unfortunately name-entangled with "app" via PreallocatedProcessManager:
* ContentParent::PreallocateAppProcess
* ContentParent::GetNewOrPreallocatedAppProcess
???
* CreateBrowserOrApp (static, takes ContentParent*, returns TabParent*)


#### When To Terminate ####
Two ways:
* Recursively, on ContentParent::ActorDestroy, posting runnables to invoke
  ShutDownProcess on each immediate child ContentParent*.  The authoritative
  list is mainted by and therefore retrieved from the ContentProcessManager.
  * (Invoked by failed construction or OnChannelClose/OnChannelError)
* ContentParent::NotifyTabDestroyed posts a runnable to ShutDownProcess when
  it gets invoked and it was the last tab.  (With ContentProcessManager again
  storing the authoritative list; it asks for all the TabIds for our
  ContentParentId.)
  * Invoked by:
    * In the parent process: (static) ContentParent::DeallocateTabId which is
      in turn called by TabParent::Recv__delete__.
    * In content processes, where a ContentBridgeParent is therefore involved,
      a ping-pong flow is required.  TabParent::Recv__delete__ still calls
      ContentParent::DeallocateTabId but it instead calls SendDeallocateTabId
      which ContentParent::RecvDeallocateTabId receives and then calls
      DeallocateTabId on itself, which is either now the right process or keeps
      making its way upward.  Really, the whole nested hierarchy is a bit
      confusing and seems sketchy from a security perspective since the values
      are blindly passed and consumed.

#### Terminating ####
ContentParent::ActorDestroy unsurprisingly is the guts.  It:
* calls ShutDownProcess
* calls ContentProcessManager::RemoveContentProcess
* schedules some Delayed* runnables for refcount and IPC IO loop reasons.

## Low Level Function Notes ##

### ContentParent::CreateBrowserOrApp ###

* Quick bail via TabParent::GetNextTabParent() check which is part of the
  AutoUseNewTab infrastructure for window.open().
* Non-app case:  Defined as (IsMozBrowserElement() || !HasOwnApp()) where
  IsMozBrowserElement explicitly is false if there's a "mozapp" attribute, and
  HasOwnApp corresponds to being part of an app.  (And HasAppOwnerApp should
  also be falsed because it requires HasOwnApp()).
    * Determines constructorSender, one of:
      * Creates a bridge if this is not the parent process.
      * Uses explicitly provided aOpenerContentParent if provided.
      * Otherwise, creates a new process via GetNewOrUsedBrowserProcess.
* App case is leftover.  It does the sAppContentParents tracking.
