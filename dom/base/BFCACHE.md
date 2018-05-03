## Big Picture ##

Windows have 2 interesting bfcache related states:
- Suspended
- Frozen

Being in bfcache means being frozen.  Freezing also results in suspending
the given window.

Beyond freezing, suspending can happen because:
- A modal dialog is being shown (nsGlobalWindowOuter::EnterModalState)
- The slow script dialog is being triggered for the window.

## Important Bits ##

nsDocShell (which has some nice comments):
* mOSHE, mLSHE: history entries for the old (being destroyed, unloaded) and new
  (still being loaded) page.
* CanSavePresentation
* CaptureState
* RestorePresentation
* RestoreFromHistory

## Flow Sketch ##

* nsDocShell::CreateContentViewer(,,aTryToSaveOldPresentation,)
  * nsDocShell::Embed
    * nsDocShell::SetupNewViewer
      * if (mSavingOldViewer), calls CaptureState()
        * nsDocShell::CaptureState() calls nsGlobalWindow::SaveWindowState()
          * nsGlobalWindow::Freeze **freeze happens**
    * nsDocShell::SetupNew
  * mSavingOldViewer = aTryToSaveOldPresentation && CanSavePresentation()
  * FirePageHideNotification(!mSavingOldViewer)
    * nsIContentViewer::PageHide invoked, where nsDocumentViewer is one.
      * nsDocument::OnPageHide called
        * DispatchPageTransition(target, "pagehide", aPersisted) invoked
          **hide event fired**
* nsDocShell::CreateContentViewer

???
* nsDocumentViewer::LoadComplete
  * Dispatches WidgetEvent(true, eLoad) where eLoad is EventNameList.h magic
    for a "load" event.
  * nsDocument::OnPageShow gets called
    * DispatchPageTransition(target, "pageshow", aPersisted) gets fired.

???

### Transcripts from tests
(gdb) bt
#0  0x00007fffe7d6a83e in nsGlobalWindow::Freeze() (this=0x7fffc2fd0c00) at /home/visbrero/rev_control/hg/mozilla-unified/dom/base/nsGlobalWindow.cpp:12198
#1  0x00007fffe7d55bc7 in nsGlobalWindow::SaveWindowState() (this=0x7fffc2e64c00) at /home/visbrero/rev_control/hg/mozilla-unified/dom/base/nsGlobalWindow.cpp:13092
#2  0x00007fffea1a0767 in nsDocShell::CaptureState() (this=this@entry=0x7fffbe490800) at /home/visbrero/rev_control/hg/mozilla-unified/docshell/base/nsDocShell.cpp:8288
#3  0x00007fffea1c6dbd in nsDocShell::SetupNewViewer(nsIContentViewer*) (this=this@entry=0x7fffbe490800, aNewViewer=aNewViewer@entry=0x7fffc0013420) at /home/visbrero/rev_control/hg/mozilla-unified/docshell/base/nsDocShell.cpp:9343
#4  0x00007fffea1d0d58 in nsDocShell::Embed(nsIContentViewer*, char const*, nsISupports*) (this=this@entry=0x7fffbe490800, aContentViewer=0x7fffc0013420, aCommand=aCommand@entry=0x7fffebd080f6 <ref21648> "", aExtraInfo=aExtraInfo@entry=0x0) at /home/visbrero/rev_control/hg/mozilla-unified/docshell/base/nsDocShell.cpp:7251
#5  0x00007fffea1d15ec in nsDocShell::CreateContentViewer(nsACString_internal const&, nsIRequest*, nsIStreamListener**) (this=0x7fffbe490800, aContentType=..., aRequest=aRequest@entry=0x7fffbec12858, aContentHandler=aContentHandler@entry=0x7fffba1a0108) at /home/visbrero/rev_control/hg/mozilla-unified/docshell/base/nsDocShell.cpp:9223
#6  0x00007fffea1d1acd in nsDSURIContentListener::DoContent(nsACString_internal const&, bool, nsIRequest*, nsIStreamListener**, bool*) (this=this@entry=0x7fffb79f2ce0, aContentType=..., aIsContentPreferred=aIsContentPreferred@entry=false, aRequest=aRequest@entry=0x7fffbec12858, aContentHandler=0x7fffba1a0108, aAbortProcess=aAbortProcess@entry=0x7fffffffb89b) at /home/visbrero/rev_control/hg/mozilla-unified/docshell/base/nsDSURIContentListener.cpp:128
#7  0x00007fffe77f18f5 in nsDocumentOpenInfo::TryContentListener(nsIURIContentListener*, nsIChannel*) (this=this@entry=0x7fffba1a00e0, aListener=0x7fffb79f2ce0, aChannel=0x7fffbec12858) at /home/visbrero/rev_control/hg/mozilla-unified/uriloader/base/nsURILoader.cpp:740
#8  0x00007fffe77f1ec8 in nsDocumentOpenInfo::DispatchContent(nsIRequest*, nsISupports*) (this=this@entry=0x7fffba1a00e0, request=request@entry=0x7fffbec12858, aCtxt=aCtxt@entry=0x0) at /home/visbrero/rev_control/hg/mozilla-unified/uriloader/base/nsURILoader.cpp:414
#9  0x00007fffe77f3269 in nsDocumentOpenInfo::OnStartRequest(nsIRequest*, nsISupports*) (this=0x7fffba1a00e0, request=0x7fffbec12858, aCtxt=0x0) at /home/visbrero/rev_control/hg/mozilla-unified/uriloader/base/nsURILoader.cpp:277
#10 0x00007fffe6f0c549 in mozilla::net::nsHttpChannel::CallOnStartRequest() (this=this@entry=0x7fffbec12800) at /home/visbrero/rev_control/hg/mozilla-unified/netwerk/protocol/http/nsHttpChannel.cpp:1320
#11 0x00007fffe6f0dcb5 in mozilla::net::nsHttpChannel::ContinueProcessNormal(nsresult) (this=this@entry=0x7fffbec12800, rv=<optimized out>, rv@entry=nsresult::NS_OK) at /home/visbrero/rev_control/hg/mozilla-unified/netwerk/protocol/http/nsHttpChannel.cpp:2401
#12 0x00007fffe6f1566c in mozilla::net::nsHttpChannel::ProcessNormal() (this=this@entry=0x7fffbec12800) at /home/visbrero/rev_control/hg/mozilla-unified/netwerk/protocol/http/nsHttpChannel.cpp:2336
#13 0x00007fffe6f16695 in mozilla::net::nsHttpChannel::ContinueProcessResponse2(nsresult) (this=this@entry=0x7fffbec12800, rv=nsresult::NS_OK, rv@entry=-2142568447) at /home/visbrero/rev_control/hg/mozilla-unified/netwerk/protocol/http/nsHttpChannel.cpp:2125
#14 0x00007fffe6f17161 in mozilla::net::nsHttpChannel::ContinueProcessResponse1() (this=this@entry=0x7fffbec12800) at /home/visbrero/rev_control/hg/mozilla-unified/netwerk/protocol/http/nsHttpChannel.cpp:2088
#15 0x00007fffe6f17797 in mozilla::net::nsHttpChannel::ProcessResponse() (this=this@entry=0x7fffbec12800) at /home/visbrero/rev_control/hg/mozilla-unified/netwerk/protocol/http/nsHttpChannel.cpp:2008
#16 0x00007fffe6f17ba0 in mozilla::net::nsHttpChannel::OnStartRequest(nsIRequest*, nsISupports*) (this=0x7fffbec12800, request=<optimized out>, ctxt=<optimized out>) at /home/visbrero/rev_control/hg/mozilla-unified/netwerk/protocol/http/nsHttpChannel.cpp:6507
#17 0x00007fffe6b3971d in nsInputStreamPump::OnStateStart() (this=this@entry=0x7fffb8cfe940) at /home/visbrero/rev_control/hg/mozilla-unified/netwerk/base/nsInputStreamPump.cpp:524
#18 0x00007fffe6b52edf in nsInputStreamPump::OnInputStreamReady(nsIAsyncInputStream*) (this=0x7fffb8cfe940, stream=<optimized out>) at /home/visbrero/rev_control/hg/mozilla-unified/netwerk/base/nsInputStreamPump.cpp:426
#19 0x00007fffe6a413e3 in nsInputStreamReadyEvent::Run() (this=0x7fffc00618d0) at /home/visbrero/rev_control/hg/mozilla-unified/xpcom/io/nsStreamUtils.cpp:96
#20 0x00007fffe6a7a315 in nsThread::ProcessNextEvent(bool, bool*) (this=0x7fffe4334300, aMayWait=<optimized out>, aResult=0x7fffffffc157) at /home/visbrero/rev_control/hg/mozilla-unified/xpcom/threads/nsThread.cpp:1261
#21 0x00007fffe6a7e083 in NS_ProcessNextEvent(nsIThread*, bool) (aThread=aThread@entry=0x7fffe4334300, aMayWait=aMayWait@entry=false) at /home/visbrero/rev_control/hg/mozilla-unified/xpcom/threads/nsThreadUtils.cpp:389
#22 0x00007fffe706fc64 in mozilla::ipc::MessagePump::Run(base::MessagePump::Delegate*) (this=0x7fffe43c01c0, aDelegate=0x7ffff6bb35c0) at /home/visbrero/rev_control/hg/mozilla-unified/ipc/glue/MessagePump.cpp:96
#23 0x00007fffe700866d in MessageLoop::RunInternal() (this=this@entry=0x7ffff6bb35c0) at /home/visbrero/rev_control/hg/mozilla-unified/ipc/chromium/src/base/message_loop.cc:238
#24 0x00007fffe7008691 in MessageLoop::RunHandler() (this=this@entry=0x7ffff6bb35c0) at /home/visbrero/rev_control/hg/mozilla-unified/ipc/chromium/src/base/message_loop.cc:231
#25 0x00007fffe7008af6 in MessageLoop::Run() (this=0x7ffff6bb35c0) at /home/visbrero/rev_control/hg/mozilla-unified/ipc/chromium/src/base/message_loop.cc:211
#26 0x00007fffe982a5fd in nsBaseAppShell::Run() (this=0x7fffd9b1e940) at /home/visbrero/rev_control/hg/mozilla-unified/widget/nsBaseAppShell.cpp:156
#27 0x00007fffea58b4a2 in nsAppStartup::Run() (this=0x7fffd8f1f380) at /home/visbrero/rev_control/hg/mozilla-unified/toolkit/components/startup/nsAppStartup.cpp:283
#28 0x00007fffea671771 in XREMain::XRE_mainRun() (this=this@entry=0x7fffffffc500) at /home/visbrero/rev_control/hg/mozilla-unified/toolkit/xre/nsAppRunner.cpp:4458
#29 0x00007fffea672850 in XREMain::XRE_main(int, char**, mozilla::BootstrapConfig const&) (this=this@entry=0x7fffffffc500, argc=argc@entry=5, argv=argv@entry=0x7fffffffd858, aConfig=...) at /home/visbrero/rev_control/hg/mozilla-unified/toolkit/xre/nsAppRunner.cpp:4635
#30 0x00007fffea672cd8 in XRE_main(int, char**, mozilla::BootstrapConfig const&) (argc=5, argv=0x7fffffffd858, aConfig=...) at /home/visbrero/rev_control/hg/mozilla-unified/toolkit/xre/nsAppRunner.cpp:4726
#31 0x00007fffea6842ff in mozilla::BootstrapImpl::XRE_main(int, char**, mozilla::BootstrapConfig const&) (this=<optimized out>, argc=<optimized out>, argv=<optimized out>, aConfig=...) at /home/visbrero/rev_control/hg/mozilla-unified/toolkit/xre/Bootstrap.cpp:45
#32 0x000000000040561d in do_main(int, char**, char**) (argc=argc@entry=5, argv=argv@entry=0x7fffffffd858, envp=envp@entry=0x7fffffffd888) at /home/visbrero/rev_control/hg/mozilla-unified/browser/app/nsBrowserApp.cpp:234
#33 0x0000000000405721 in main(int, char**, char**) (argc=5, argv=0x7fffffffd858, envp=0x7fffffffd888) at /home/visbrero/rev_control/hg/mozilla-unified/browser/app/nsBrowserApp.cpp:305

(gdb) cont
Continuing.
window frozen
++DOMWINDOW == 24 (0x7fffc86fa800) [pid = 22691] [serial = 24] [outer = 0x7fffc2e64c00]
TEST-PASS | /tests/dom/tests/mochitest/localstorage/test_localStorage_bfcache.html | freezing
TEST-PASS | /tests/dom/tests/mochitest/localstorage/test_localStorage_bfcache.html | mutated

Thread 1 "firefox" hit Breakpoint 1, nsGlobalWindow::Freeze (this=0x7fffc86fa800) at /home/visbrero/rev_control/hg/mozilla-unified/dom/base/nsGlobalWindow.cpp:12198
12198	{
(gdb) bt
#0  0x00007fffe7d6a83e in nsGlobalWindow::Freeze() (this=0x7fffc86fa800) at /home/visbrero/rev_control/hg/mozilla-unified/dom/base/nsGlobalWindow.cpp:12198
#1  0x00007fffe7d55bc7 in nsGlobalWindow::SaveWindowState() (this=0x7fffc2e64c00) at /home/visbrero/rev_control/hg/mozilla-unified/dom/base/nsGlobalWindow.cpp:13092
#2  0x00007fffea1a0767 in nsDocShell::CaptureState() (this=this@entry=0x7fffbe490800) at /home/visbrero/rev_control/hg/mozilla-unified/docshell/base/nsDocShell.cpp:8288
#3  0x00007fffea1e059e in nsDocShell::RestoreFromHistory() (this=0x7fffbe490800) at /home/visbrero/rev_control/hg/mozilla-unified/docshell/base/nsDocShell.cpp:8656
#4  0x00007fffea1e1cae in nsDocShell::RestorePresentationEvent::Run() (this=<optimized out>) at /home/visbrero/rev_control/hg/mozilla-unified/docshell/base/nsDocShell.cpp:8334
#5  0x00007fffe6a7a315 in nsThread::ProcessNextEvent(bool, bool*) (this=0x7fffe4334300, aMayWait=<optimized out>, aResult=0x7fffffffc157) at /home/visbrero/rev_control/hg/mozilla-unified/xpcom/threads/nsThread.cpp:1261
#6  0x00007fffe6a7e083 in NS_ProcessNextEvent(nsIThread*, bool) (aThread=aThread@entry=0x7fffe4334300, aMayWait=aMayWait@entry=false) at /home/visbrero/rev_control/hg/mozilla-unified/xpcom/threads/nsThreadUtils.cpp:389
#7  0x00007fffe706fc64 in mozilla::ipc::MessagePump::Run(base::MessagePump::Delegate*) (this=0x7fffe43c01c0, aDelegate=0x7ffff6bb35c0) at /home/visbrero/rev_control/hg/mozilla-unified/ipc/glue/MessagePump.cpp:96
#8  0x00007fffe700866d in MessageLoop::RunInternal() (this=this@entry=0x7ffff6bb35c0) at /home/visbrero/rev_control/hg/mozilla-unified/ipc/chromium/src/base/message_loop.cc:238
#9  0x00007fffe7008691 in MessageLoop::RunHandler() (this=this@entry=0x7ffff6bb35c0) at /home/visbrero/rev_control/hg/mozilla-unified/ipc/chromium/src/base/message_loop.cc:231
#10 0x00007fffe7008af6 in MessageLoop::Run() (this=0x7ffff6bb35c0) at /home/visbrero/rev_control/hg/mozilla-unified/ipc/chromium/src/base/message_loop.cc:211
#11 0x00007fffe982a5fd in nsBaseAppShell::Run() (this=0x7fffd9b1e940) at /home/visbrero/rev_control/hg/mozilla-unified/widget/nsBaseAppShell.cpp:156
#12 0x00007fffea58b4a2 in nsAppStartup::Run() (this=0x7fffd8f1f380) at /home/visbrero/rev_control/hg/mozilla-unified/toolkit/components/startup/nsAppStartup.cpp:283
#13 0x00007fffea671771 in XREMain::XRE_mainRun() (this=this@entry=0x7fffffffc500) at /home/visbrero/rev_control/hg/mozilla-unified/toolkit/xre/nsAppRunner.cpp:4458
#14 0x00007fffea672850 in XREMain::XRE_main(int, char**, mozilla::BootstrapConfig const&) (this=this@entry=0x7fffffffc500, argc=argc@entry=5, argv=argv@entry=0x7fffffffd858, aConfig=...) at /home/visbrero/rev_control/hg/mozilla-unified/toolkit/xre/nsAppRunner.cpp:4635
#15 0x00007fffea672cd8 in XRE_main(int, char**, mozilla::BootstrapConfig const&) (argc=5, argv=0x7fffffffd858, aConfig=...) at /home/visbrero/rev_control/hg/mozilla-unified/toolkit/xre/nsAppRunner.cpp:4726
#16 0x00007fffea6842ff in mozilla::BootstrapImpl::XRE_main(int, char**, mozilla::BootstrapConfig const&) (this=<optimized out>, argc=<optimized out>, argv=<optimized out>, aConfig=...) at /home/visbrero/rev_control/hg/mozilla-unified/toolkit/xre/Bootstrap.cpp:45
#17 0x000000000040561d in do_main(int, char**, char**) (argc=argc@entry=5, argv=argv@entry=0x7fffffffd858, envp=envp@entry=0x7fffffffd888) at /home/visbrero/rev_control/hg/mozilla-unified/browser/app/nsBrowserApp.cpp:234
#18 0x0000000000405721 in main(int, char**, char**) (argc=5, argv=0x7fffffffd858, envp=0x7fffffffd888) at /home/visbrero/rev_control/hg/mozilla-unified/browser/app/nsBrowserApp.cpp:305
(gdb) cont
Continuing.
window frozen
TEST-PASS | /tests/dom/tests/mochitest/localstorage/test_localStorage_bfcache.html | unfrozen

(gdb) bt
#0  0x00007fffe7d6aac0 in nsGlobalWindow::Thaw() (this=0x7fffbdbd2400) at /home/visbrero/rev_control/hg/mozilla-unified/dom/base/nsGlobalWindow.cpp:12229
#1  0x00007fffe7d8acf7 in nsGlobalWindow::RestoreWindowState(nsISupports*) (this=<optimized out>, aState=<optimized out>) at /home/visbrero/rev_control/hg/mozilla-unified/dom/base/nsGlobalWindow.cpp:13135
#2  0x00007fffea1e1590 in nsDocShell::RestoreFromHistory() (this=0x7fffb7a74000) at /home/visbrero/rev_control/hg/mozilla-unified/docshell/base/nsDocShell.cpp:8928
#3  0x00007fffea1e1cae in nsDocShell::RestorePresentationEvent::Run() (this=<optimized out>) at /home/visbrero/rev_control/hg/mozilla-unified/docshell/base/nsDocShell.cpp:8334
#4  0x00007fffe6a7a315 in nsThread::ProcessNextEvent(bool, bool*) (this=0x7fffe4334300, aMayWait=<optimized out>, aResult=0x7fffffffc157) at /home/visbrero/rev_control/hg/mozilla-unified/xpcom/threads/nsThread.cpp:1261
#5  0x00007fffe6a7e083 in NS_ProcessNextEvent(nsIThread*, bool) (aThread=aThread@entry=0x7fffe4334300, aMayWait=aMayWait@entry=false) at /home/visbrero/rev_control/hg/mozilla-unified/xpcom/threads/nsThreadUtils.cpp:389
#6  0x00007fffe706fc64 in mozilla::ipc::MessagePump::Run(base::MessagePump::Delegate*) (this=0x7fffe43c01c0, aDelegate=0x7ffff6bb35c0) at /home/visbrero/rev_control/hg/mozilla-unified/ipc/glue/MessagePump.cpp:96
#7  0x00007fffe700866d in MessageLoop::RunInternal() (this=this@entry=0x7ffff6bb35c0) at /home/visbrero/rev_control/hg/mozilla-unified/ipc/chromium/src/base/message_loop.cc:238
#8  0x00007fffe7008691 in MessageLoop::RunHandler() (this=this@entry=0x7ffff6bb35c0) at /home/visbrero/rev_control/hg/mozilla-unified/ipc/chromium/src/base/message_loop.cc:231
#9  0x00007fffe7008af6 in MessageLoop::Run() (this=0x7ffff6bb35c0) at /home/visbrero/rev_control/hg/mozilla-unified/ipc/chromium/src/base/message_loop.cc:211
#10 0x00007fffe982a5fd in nsBaseAppShell::Run() (this=0x7fffd9b1e940) at /home/visbrero/rev_control/hg/mozilla-unified/widget/nsBaseAppShell.cpp:156
#11 0x00007fffea58b4a2 in nsAppStartup::Run() (this=0x7fffd8f1f4c0) at /home/visbrero/rev_control/hg/mozilla-unified/toolkit/components/startup/nsAppStartup.cpp:283
#12 0x00007fffea671771 in XREMain::XRE_mainRun() (this=this@entry=0x7fffffffc500) at /home/visbrero/rev_control/hg/mozilla-unified/toolkit/xre/nsAppRunner.cpp:4458
#13 0x00007fffea672850 in XREMain::XRE_main(int, char**, mozilla::BootstrapConfig const&) (this=this@entry=0x7fffffffc500, argc=argc@entry=5, argv=argv@entry=0x7fffffffd858, aConfig=...) at /home/visbrero/rev_control/hg/mozilla-unified/toolkit/xre/nsAppRunner.cpp:4635
#14 0x00007fffea672cd8 in XRE_main(int, char**, mozilla::BootstrapConfig const&) (argc=5, argv=0x7fffffffd858, aConfig=...) at /home/visbrero/rev_control/hg/mozilla-unified/toolkit/xre/nsAppRunner.cpp:4726
#15 0x00007fffea6842ff in mozilla::BootstrapImpl::XRE_main(int, char**, mozilla::BootstrapConfig const&) (this=<optimized out>, argc=<optimized out>, argv=<optimized out>, aConfig=...) at /home/visbrero/rev_control/hg/mozilla-unified/toolkit/xre/Bootstrap.cpp:45
#16 0x000000000040561d in do_main(int, char**, char**) (argc=argc@entry=5, argv=argv@entry=0x7fffffffd858, envp=envp@entry=0x7fffffffd888) at /home/visbrero/rev_control/hg/mozilla-unified/browser/app/nsBrowserApp.cpp:234
#17 0x0000000000405721 in main(int, char**, char**) (argc=5, argv=0x7fffffffd858, envp=0x7fffffffd888) at /home/visbrero/rev_control/hg/mozilla-unified/browser/app/nsBrowserApp.cpp:305
