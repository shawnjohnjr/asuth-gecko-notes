https://bugzilla.mozilla.org/show_bug.cgi?id=1383518

Upload breaks, baku failed to repro, repro complicated by account necessities,
and the reporter very helpfully bisected down to the commit, so going with patch
inspection first.  (Noting that extensive refactoring of the blob codebase has
happenened since the patch, so the focus is on determining the implied
conditions in order to then reproduce them in a test.)

## Bug 1393063 variant (built against release) ##

### Investigating the stream that the HttpChannelParent deserializes

(rr) break HttpChannelParent.cpp:570


### Messed up File closing

(more recent investigation at top, older below)

#### Source of the close

This just looks like REOPEN_ON_REWIND is freaking out.

The PR_Close invocation line:
```
(rr) break nsFileStreams.cpp:169
Breakpoint 2 at 0x7f1306e66aae: file /home/visbrero/rev_control/hg/mozilla-beta/netwerk/base/nsFileStreams.cpp, line 169.
```

```
Thread 5 hit Breakpoint 2, nsFileStreamBase::Close (this=this@entry=0x7f12d5fee860) at /home/visbrero/rev_control/hg/mozilla-beta/netwerk/base/nsFileStreams.cpp:169
169	        if (PR_Close(mFD) == PR_FAILURE)
(rr) cbt
  0 nsFileStreamBase::Close                     at netwerk/base/nsFileStreams.cpp:169 0x7f1306e66aae
      this=0x7f12d5fee860 this0
  1 nsFileInputStream::Close                    at netwerk/base/nsFileStreams.cpp:495 0x7f1306e6bc8d
      this=0x7f12d5fee860 this0
  2 nsFileInputStream::Serialize                at netwerk/base/nsFileStreams.cpp:602 0x7f1306e75f5c
      this=0x7f12d5fee860 this0
      aParams=@0x7f12eb2fd790
      aFileDescriptors=nsIIPCSerializableInputStream::FileDescriptorArray & = {{     mHandle = 84   }}
  3 ipc::(anonymous namespace)::SerializeInputStreamWithFdsParent<mozilla::ipc::PBackgroundParent> at ipc/glue/IPCStreamUtils.cpp:106 0x7f13073f3b09
      aStream=0x7f12d5fee8b8
      aValue=@0x7f12eb2fd790
      aManager=0x7f12e3fe3800 aManager3
  4 ipc::(anonymous namespace)::SerializeInputStreamParent<mozilla::ipc::PBackgroundParent> at ipc/glue/IPCStreamUtils.cpp:239 0x7f13073fa47f
      aStream=0x7f12d5fee8a8 aStream4
      aManager=0x7f12e3fe3800 aManager3
      aValue=0x0 aValue4
      aOptionalValue=0x7f12eb2fd790 aOptionalValue4
      aDelayedStart=<optimized out>
  5 ipc::AutoIPCStream::Serialize               at ipc/glue/IPCStreamUtils.cpp:570 0x7f13073fa639
      this=0x7f12eb2fd790 aOptionalValue4
      aStream=0x7f12d5fee8a8 aStream4
      aManager=0x7f12e3fe3800 aManager3
  6 dom::IPCBlobInputStreamParent::RecvStreamNeeded at dom/file/ipc/IPCBlobInputStreamParent.cpp:129 0x7f130927b296
      this=0x7f12d609a340 this6
  7 ipc::PIPCBlobInputStreamParent::OnMessageReceived at obj-firefox-debug/ipc/ipdl/PIPCBlobInputStreamParent.cpp:110 0x7f1307529cc9
      this=0x7f12d609a340 this6
      msg__=<optimized out>
  8 ipc::PBackgroundParent::OnMessageReceived   at obj-firefox-debug/ipc/ipdl/PBackgroundParent.cpp:1006 0x7f130787713a
      this=0x7f12e3fe3800 aManager3
      msg__=@0x7f12e43c4cb0
  9 ipc::MessageChannel::DispatchAsyncMessage   at ipc/glue/MessageChannel.cpp:2075 0x7f13073f0161
      this=0x7f12e3fe3920 this9
      aMsg=@0x7f12e43c4cb0
 10 ipc::MessageChannel::DispatchMessage(IPC::Message&&) at ipc/glue/MessageChannel.cpp:2001 0x7f13073fd005

 11 ipc::MessageChannel::RunMessage             at ipc/glue/MessageChannel.cpp:1870 0x7f13073ff28a
      this=0x7f12e3fe3920 this9
      aTask=@0x7f12e43c4c60
 12 ipc::MessageChannel::MessageTask::Run       at ipc/glue/MessageChannel.cpp:1903 0x7f13073ff4f3
      this=0x7f12e43c4c60
 13 nsThread::ProcessNextEvent                  at xpcom/threads/nsThread.cpp:1418 0x7f1306da0448
      this=0x7f12f9d614c0 this13
      aMayWait=<optimized out>
      aResult=0x7f12eb2fdd27
 14 NS_ProcessNextEvent                         at xpcom/threads/nsThreadUtils.cpp:475 0x7f1306da44b2
      aThread=0x7f12f9d614c0 this13
      aMayWait=false
 15 ipc::MessagePumpForNonMainThreads::Run      at ipc/glue/MessagePump.cpp:338 0x7f13073f4952
      this=0x7f12f29f1980
      aDelegate=0x7f12ebe27330 aDelegate15
 16 MessageLoop::RunInternal                    at ipc/chromium/src/base/message_loop.cc:238 0x7f13073727ad
      this=0x7f12ebe27330 aDelegate15
 17 MessageLoop::RunHandler                     at ipc/chromium/src/base/message_loop.cc:231 0x7f13073727f9
      this=0x7f12ebe27330 aDelegate15
 18 MessageLoop::Run                            at ipc/chromium/src/base/message_loop.cc:211 0x7f1307372a0c
      this=0x7f12ebe27330 aDelegate15
 19 nsThread::ThreadFunc                        at xpcom/threads/nsThread.cpp:501 0x7f1306da4781
      aArg=<optimized out>
 20 _pt_root                                    at nsprpub/pr/src/pthreads/ptthread.c:216 0x7f131a2bf6e3
      arg=0x7f12f3650aa0 arg20
 21 start_thread                                at pthread_create.c:456 0x7f1319dfe6da
      arg=0x7f12eb2fe700 arg21
 22 clone                                       at ???:105 0x7f131908cd7f
```

#### Parent NS_BASE_STREAM_CLOSED generation

```
(rr) cbt
  0 nsFileStreamBase::DoPendingOpen             at netwerk/base/nsFileStreams.cpp:374 0x7f1306e95e51
      this=0x7f12d5fee860 this0
  1 nsFileStreamBase::Tell                      at netwerk/base/nsFileStreams.cpp:73 0x7f1306e6bbb7
      this=0x7f12d5fee860 this0
      result=0x7f12d5fee8e0
  2 nsFileInputStream::Close                    at netwerk/base/nsFileStreams.cpp:490 0x7f1306e6bcc0
      this=0x7f12d5fee860 this0
  3 nsFileInputStream::~nsFileInputStream       at netwerk/base/nsFileStreams.h:161 0x7f1306e962ee
      this=0x7f12d5fee860 this0
      __in_chrg=<optimized out>
  4 nsFileInputStream::~nsFileInputStream       at netwerk/base/nsFileStreams.h:162 0x7f1306e962ee
      this=0x7f12d5fee860 this0
      __in_chrg=<optimized out>
  5 nsFileStreamBase::Release                   at netwerk/base/nsFileStreams.cpp:55 0x7f1306e6598d
      this=0x7f12d5fee860 this0
  6 nsFileInputStream::Release                  at netwerk/base/nsFileStreams.cpp:389 0x7f1306e659ad
      this=0x7f12d5fee860 this0
  7 nsCOMPtr<nsIInputStream>::~nsCOMPtr         at obj-firefox-debug/dist/include/nsCOMPtr.h:404 0x7f130927b1a2
      this=0x7f12eb2fd738
      __in_chrg=<optimized out>
  8 dom::IPCBlobInputStreamParent::RecvStreamNeeded at dom/file/ipc/IPCBlobInputStreamParent.cpp:111 0x7f130927b1a2
      this=<optimized out>
  9 ipc::PIPCBlobInputStreamParent::OnMessageReceived at obj-firefox-debug/ipc/ipdl/PIPCBlobInputStreamParent.cpp:110 0x7f1307529cc9
      this=0x7f12d609a340
      msg__=<optimized out>
 10 ipc::PBackgroundParent::OnMessageReceived   at obj-firefox-debug/ipc/ipdl/PBackgroundParent.cpp:1006 0x7f130787713a
      this=0x7f12e3fe3800
      msg__=@0x7f12e43c4cb0
 11 ipc::MessageChannel::DispatchAsyncMessage   at ipc/glue/MessageChannel.cpp:2075 0x7f13073f0161
      this=0x7f12e3fe3920 this11
      aMsg=@0x7f12e43c4cb0
 12 ipc::MessageChannel::DispatchMessage(IPC::Message&&) at ipc/glue/MessageChannel.cpp:2001 0x7f13073fd005

 13 ipc::MessageChannel::RunMessage             at ipc/glue/MessageChannel.cpp:1870 0x7f13073ff28a
      this=0x7f12e3fe3920 this11
      aTask=@0x7f12e43c4c60
 14 ipc::MessageChannel::MessageTask::Run       at ipc/glue/MessageChannel.cpp:1903 0x7f13073ff4f3
      this=0x7f12e43c4c60
 15 nsThread::ProcessNextEvent                  at xpcom/threads/nsThread.cpp:1418 0x7f1306da0448
      this=0x7f12f9d614c0 this15
      aMayWait=<optimized out>
      aResult=0x7f12eb2fdd27
 16 NS_ProcessNextEvent                         at xpcom/threads/nsThreadUtils.cpp:475 0x7f1306da44b2
      aThread=0x7f12f9d614c0 this15
      aMayWait=false
 17 ipc::MessagePumpForNonMainThreads::Run      at ipc/glue/MessagePump.cpp:338 0x7f13073f4952
      this=0x7f12f29f1980
      aDelegate=0x7f12ebe27330 aDelegate17
 18 MessageLoop::RunInternal                    at ipc/chromium/src/base/message_loop.cc:238 0x7f13073727ad
      this=0x7f12ebe27330 aDelegate17
 19 MessageLoop::RunHandler                     at ipc/chromium/src/base/message_loop.cc:231 0x7f13073727f9
      this=0x7f12ebe27330 aDelegate17
 20 MessageLoop::Run                            at ipc/chromium/src/base/message_loop.cc:211 0x7f1307372a0c
      this=0x7f12ebe27330 aDelegate17
 21 nsThread::ThreadFunc                        at xpcom/threads/nsThread.cpp:501 0x7f1306da4781
      aArg=<optimized out>
 22 _pt_root                                    at nsprpub/pr/src/pthreads/ptthread.c:216 0x7f131a2bf6e3
      arg=0x7f12f3650aa0 arg22
 23 start_thread                                at pthread_create.c:456 0x7f1319dfe6da
      arg=0x7f12eb2fe700 arg23
 24 clone                                       at ???:105 0x7f131908cd7f
```


#### Suspicion
The I/O log showed right after the ContinueAsyncOpen in the child running:
```
[Parent 22222] WARNING: NS_ENSURE_SUCCESS(rv, rv) failed with result 0x80470002: file /home/visbrero/rev_control/hg/mozilla-beta/netwerk/base/nsFileStreams.cpp, line 74
```
which is NS_BASE_STREAM_CLOSED
```
(rr) pp (nsresult)0x80470002
nsresult::NS_BASE_STREAM_CLOSED
```


### Breakpoints
#### Child
```
(rr) info breakpoints
Num     Type           Disp Enb Address    What
1       breakpoint     keep y   <PENDING>  HttpChannelChild.cpp:2463
2       breakpoint     keep y   <PENDING>  HttpChannelChild.cpp:2464

(rr) break mozilla::ipc::IPCStreamSource::OnEnd(nsresult)
Breakpoint 4 at 0x7fab4c815fda: file /home/visbrero/rev_control/hg/mozilla-beta/ipc/glue/IPCStreamSource.cpp, line 298.
```
#### Parent ####
HttpChannelChild::OnCopyComplete

## Alternate Repro ##

Youtube thumbnail upload.

### Breakpoint says:

e, where e is a form with 5 inputs.  e[0] is the file input.  e[1-4] are hidden
inputs with string values.

e.action
"https://www.youtube.com/my_thumbnail_post"
e.encoding
"multipart/form-data"

"<form method=\"post\" enctype=\"multipart/form-data\" action=\"https://www.youtube.com/my_thumbnail_post\" target=\"closure_frame1_0_inner\"><input style=\"position: absolute; right: 0px; top: 0px; cursor: pointer; opacity: 0; font-size: 25px;\" tabindex=\"-1\" name=\"imagefile\" type=\"file\"><input name=\"video_id\" value=\"Xg3gfm3Fu14\" type=\"hidden\"><input name=\"is_ajax\" value=\"1\" type=\"hidden\"><input name=\"o\" value=\"U\" type=\"hidden\"><input name=\"session_token\" value=\"QUFFLUhqbnBTSWZ5TFZEalJhOUljUHg0TGVOcnNEZmhiUXxBQ3Jtc0ttNUtRQzFiY1VyOFNRVlR3MXp4TkNMc0FFTTV4alRkUFZBMUZ0Ui05MXdpNGhRb21qV1B4R2FXTmhJWWRYa2xaS1Z6WWR1QjNCQmNPN0ZHOEM5dWQ2YnFXcTFfU3pGQmE0UXpkX0Y0OXNISmtmYXljd3BWWVdzVEdrYkhNZVU1T3h4LXpQeXZ1T0N3UnRob2FNQW9YRXY0ZXZjOWc=\" type=\"hidden\"></form>"

window.frames[3]
Window https://www.youtube.com/edit?o=U&video_id=Xg3gfm3Fu14

window.frames[3].name
"closure_frame1_0"

window.frames[3].frames[0].name
"closure_frame1_0_inner"

### globals weirdness / maybe importNode complexities ###

There's some weirdness with the globals and such:

e.ownerDocument.defaultView.parent === window
true

Code seems to be:
```
function Xr(a) {
a.yb = !0;
a.jc = 0;
a.qc = a.f + '_' + (a.zm++).toString(36);
a.ya = bg(a.ga).ja('IFRAME', {
  name: a.qc,
  id: a.qc
});
E && 7 > Number(Sc) && (a.ya.src = 'javascript:""');
var b = a.ya.style;
b.visibility = 'hidden';
b.width = b.height = '10px';
b.display = 'none';
Ec ? b.marginTop = b.marginLeft = '-10px' : (b.position = 'absolute', b.top = b.left = '-10px');
if (E && !F('11')) {
  a.ga.target = a.qc || '';
  bg(a.ga).b.body.appendChild(a.ya);
  L(a.ya, 'readystatechange', a.mg, !1, a);
  try {
    a.b = !1,
    a.ga.submit()
  } catch (Fv) {
    Kh(a.ya, 'readystatechange', a.mg, !1, a),
    cs(a, 1)
  }
} else {
  bg(a.ga).b.body.appendChild(a.ya);
  b = a.qc + '_inner';
  var c = Kg(a.ya);
  if (document.baseURI) {
    var d = Ua(b);
    d = Rf(cf('Short HTML snippet, input escaped, safe URL, for performance'), '<head><base href="' + Ua(document.baseURI) + '"></head><body><iframe id="' + d + '" name="' + d + '"></iframe>')
  } else d = Ua(b),
    d = Rf(cf('Short HTML snippet, input escaped, for performance'), '<body><iframe id="' + d + '" name="' + d + '"></iframe>');
  Ac && !Ec ? c.documentElement.innerHTML = Ff(d)  : c.write(Ff(d));
  L(c.getElementById(b), 'load', a.Ie, !1, a);
  var e = fg('TEXTAREA', a.ga);
  d = 0;
  for (var f = e.length; d < f; d++) {
    var h = e[d].value;
    Yg(e[d]) != h && (K(e[d], h), e[d].value = h)
  }
  e = c.importNode(a.ga, !0);
  e.target = b;
  e.action = a.ga.action;
  c.body.appendChild(e);
  h = fg('SELECT', a.ga);
  var k = fg('SELECT', e);
  d = 0;
  for (f = h.length; d < f; d++) for (var l = fg('OPTION', h[d]), m = fg('OPTION', k[d]), p = 0, t = l.length; p < t; p++) m[p].selected = l[p].selected;
  h = fg('INPUT', a.ga);
  k = fg('INPUT', e);
  d = 0;
  for (f = h.length; d < f; d++) if ('file' == h[d].type && h[d].value != k[d].value) {
    a.ga.target = b;
    e = a.ga;
    break
  }
  try {
  a.b =
  !1,
  e.submit(),
```
Where:
* fg(tagName, elem) is getElementsByTagName with doc assumed if not elem.
* bg(a) is an aggregate: given a node, it creates a new document  forked off
  from the given node.
  * dg(a) normalizes to the document of a window, or a document's parent (aka
    ownerDocument)
  * cg(a) [which assumes n is the global window], uses 'a' if provided,
    otherwise it news up an HTMLDocument.  Either way it saves the result to b,
    so when used with new you get { b: HTMLDocument }.
* Kg(a) normalizes to be the contentDocument of the given iframe.
* Caller Zr(a, b) took a b which seems to be the actual prototype form.  Zr
  uses some kind of sanitizing escaping on the original action.  Its initial
  payload from the page itself is:
  b.outerHTML
"<form method=\"post\" enctype=\"multipart/form-data\" action=\"https://www.youtube.com/my_thumbnail_post\"><input style=\"position: absolute; right: 0px; top: 0px; cursor: pointer; opacity: 0; font-size: 25px;\" tabindex=\"-1\" name=\"imagefile\" type=\"file\"><input name=\"video_id\" value=\"Xg3gfm3Fu14\" type=\"hidden\"><input name=\"is_ajax\" value=\"1\" type=\"hidden\"><input name=\"o\" value=\"U\" type=\"hidden\"><input name=\"session_token\" value=\"QUFFLUhqbnBTSWZ5TFZEalJhOUljUHg0TGVOcnNEZmhiUXxBQ3Jtc0ttNUtRQzFiY1VyOFNRVlR3MXp4TkNMc0FFTTV4alRkUFZBMUZ0Ui05MXdpNGhRb21qV1B4R2FXTmhJWWRYa2xaS1Z6WWR1QjNCQmNPN0ZHOEM5dWQ2YnFXcTFfU3pGQmE0UXpkX0Y0OXNISmtmYXljd3BWWVdzVEdrYkhNZVU1T3h4LXpQeXZ1T0N3UnRob2FNQW9YRXY0ZXZjOWc=\" type=\"hidden\"></form>"
* Xr's `a.ga` is that raw form.


The analysis of interesting stuff seems to be:
* a.ya seems to be a created iframe?
* then c is its contentDocument
* HTML is forced into the document.
* Any TEXTAREA is flattened, it looks like.
* e is created by invoking c.importNode(a.ga [the form], deep=true)
* e is then appended to c.body.
* Some propagation passes occur:
  *

## rr repro 2 analysis
(divergence happened due to reboots/updates, had to ditch repro 1)

### canonical open meta

(rr) when
Current event: 186970

### breakpoints
added breakpoint 3 to back up to the SW failure to paranoia check that one.
```
(rr) info breakpoints
Num     Type           Disp Enb Address            What
1       breakpoint     keep y   0x00007f0d31467ad7 in mozilla::net::HttpChannelChild::ContinueAsyncOpen() at /home/visbrero/rev_control/hg/mozilla-central/netwerk/protocol/http/HttpChannelChild.cpp:2523
	breakpoint already hit 3 times
2       breakpoint     keep y   0x00007f0d31467af6 in mozilla::net::HttpChannelChild::ContinueAsyncOpen() at /home/visbrero/rev_control/hg/mozilla-central/netwerk/protocol/http/HttpChannelChild.cpp:2524
	breakpoint already hit 2 times
3       breakpoint     keep y   0x00007f0d33df61c2 in mozilla::dom::workers::(anonymous namespace)::ContinueDispatchFetchEventRunnable::Run() at /home/visbrero/rev_control/hg/mozilla-central/dom/workers/ServiceWorkerManager.cpp:2712
```
#### SW re-investigation, breakpoint 3



```
(rr) break 'mozilla::net::HttpChannelChild::OnCopyComplete'
Breakpoint 10 at 0x7f0d3144bee2: file /home/visbrero/rev_control/hg/mozilla-central/netwerk/protocol/http/HttpChannelChild.cpp, line 3397.
```

right, so the problem was OnCopyComplete was shadowed by the child, whoops.

```
(rr) p &channel->mStatus.mValue._M_i
$9 = (nsresult *) 0x7f0d163833d0
(rr) watch *0x7f0d163833d0
Hardware watchpoint 8: *0x7f0d163833d0
```

er, the below breakpoint didn't work, switching to data breakpoint

```
(rr) break 'mozilla::net::HttpBaseChannel::OnCopyComplete(nsresult)'
Breakpoint 4 at 0x7f0d313ffcdc: file /home/visbrero/rev_control/hg/mozilla-central/netwerk/protocol/http/HttpBaseChannel.cpp, line 916.
```

The problem is an NS_BASE_STREAM_WOULD_BLOCK being returned as the
HttpBaseChannel::OnCopyComplete result of HttpBaseChannel::EnsureUploadStreamIsCloneable

```
(rr) cbt paste
  0 dom::workers::(anonymous namespace)::ContinueDispatchFetchEventRunnable::Run
    dom/workers/ServiceWorkerManager.cpp:2712
  1 nsPermissionManager::WhenPermissionsAvailable
    extensions/cookie/nsPermissionManager.cpp:3368
  2 dom::workers::ServiceWorkerManager::<lambda()>::operator()
    dom/workers/ServiceWorkerManager.cpp:2819
  3 detail::RunnableFunction<mozilla::dom::workers::ServiceWorkerManager::DispatchFetchEvent(const mozilla::OriginAttributes&, nsIDocument*, const nsAString&, nsIInterceptedChannel*, bool, bool, mozilla::ErrorResult&)::<lambda()> >::Run(void)
    obj-firefox-debug/dist/include/nsThreadUtils.h:527
  4 net::HttpBaseChannel::EnsureUploadStreamIsCloneableComplete
    netwerk/protocol/http/HttpBaseChannel.cpp:939
  5 detail::RunnableMethodArguments<nsresult>::applyImpl<mozilla::net::HttpChannelChild, void (mozilla::net::HttpBaseChannel::*)(nsresult), StoreCopyPassByConstLRef<nsresult>, 0ul>
    obj-firefox-debug/dist/include/nsThreadUtils.h:1142
  6 detail::RunnableMethodArguments<nsresult>::apply<mozilla::net::HttpChannelChild, void (mozilla::net::HttpBaseChannel::*)(nsresult)>
    obj-firefox-debug/dist/include/nsThreadUtils.h:1149
  7 detail::RunnableMethodImpl<mozilla::net::HttpChannelChild*, void (mozilla::net::HttpBaseChannel::*)(nsresult), true, (mozilla::RunnableKind)0, nsresult>::Run
    obj-firefox-debug/dist/include/nsThreadUtils.h:1192
  8 nsThread::ProcessNextEvent
    xpcom/threads/nsThread.cpp:1033
  9 NS_ProcessNextEvent
    xpcom/threads/nsThreadUtils.cpp:521
 10 ipc::MessagePump::Run
    ipc/glue/MessagePump.cpp:97
 11 ipc::MessagePumpForChildProcess::Run
    ipc/glue/MessagePump.cpp:302
 12 MessageLoop::RunInternal
    ipc/chromium/src/base/message_loop.cc:326
 13 MessageLoop::RunHandler
    ipc/chromium/src/base/message_loop.cc:319
 14 MessageLoop::Run
    ipc/chromium/src/base/message_loop.cc:299
 15 nsBaseAppShell::Run
    widget/nsBaseAppShell.cpp:158
 16 XRE_RunAppShell
    toolkit/xre/nsEmbedFunctions.cpp:882
 17 ipc::MessagePumpForChildProcess::Run
    ipc/glue/MessagePump.cpp:270
 18 MessageLoop::RunInternal
    ipc/chromium/src/base/message_loop.cc:326
 19 MessageLoop::RunHandler
    ipc/chromium/src/base/message_loop.cc:319
 20 MessageLoop::Run
    ipc/chromium/src/base/message_loop.cc:299
 21 XRE_InitChildProcess
    toolkit/xre/nsEmbedFunctions.cpp:699
 22 BootstrapImpl::XRE_InitChildProcess
    toolkit/xre/Bootstrap.cpp:65
 23 content_process_main
    browser/app/../../ipc/contentproc/plugin-container.cpp:64
 24 main
    browser/app/nsBrowserApp.cpp:285
```

## rr repro 1 analysis

### breakpoints:
(rr) info break
Num     Type           Disp Enb Address            What
1       breakpoint     keep y   0x00007f2e10b12065 in mozilla::net::HttpChannelChild::ContinueAsyncOpen() at /home/visbrero/rev_control/hg/mozilla-unified/netwerk/protocol/http/HttpChannelChild.cpp:2506
	breakpoint already hit 4 times

#### tripped 4th time

```
Thread 1 hit Breakpoint 1, mozilla::net::HttpChannelChild::ContinueAsyncOpen (this=this@entry=0x7f2de9984800) at /home/visbrero/rev_control/hg/mozilla-unified/netwerk/protocol/http/HttpChannelChild.cpp:2506
2506	    autoStream.Serialize(mUploadStream, ContentChild::GetSingleton());
(rr) when
Current event: 230754
(rr) pp this
mozilla::net::HttpChannelChild 0x7f2de9984800
  overview:
    mRequestHead:
      mMethod: "POST"
    mURI:
      mozilla::net::nsStandardURL 0x7f2dea62e6a0
        mSpec:
          "https://www.youtube.com/my_thumbnail_post"
  context:
    mOriginalURI:
      mozilla::net::nsStandardURL 0x7f2dea62e6a0
        mSpec:
          "https://www.youtube.com/my_thumbnail_post"
    mDocumentURI:
      mozilla::net::nsStandardURL 0x7f2dea62e6a0
        mSpec:
          "https://www.youtube.com/my_thumbnail_post"
  load:
    mUploadStream:
      No mapping or pretty-printer for nsStorageInputStream, switching to gdb print.
      {
        <nsIInputStream> = {
          <nsISupports> = {
            _vptr.nsISupports = 0x7f2e189a96c0 <vtable for nsStorageInputStream+16>
          }, <No data fields>},
        <nsISeekableStream> = {
          <nsISupports> = {
            _vptr.nsISupports = 0x7f2e189a9750 <vtable for nsStorageInputStream+160>
          }, <No data fields>},
        <nsIIPCSerializableInputStream> = {
          <nsISupports> = {
            _vptr.nsISupports = 0x7f2e189a9790 <vtable for nsStorageInputStream+224>
          }, <No data fields>},
        <nsICloneableInputStream> = {
          <nsISupports> = {
            _vptr.nsISupports = 0x7f2e189a97d0 <vtable for nsStorageInputStream+288>
          }, <No data fields>},
        members of nsStorageInputStream:
        mRefCnt = {
          static isThreadSafe = true,
          mValue = {
            <std::__atomic_base<unsigned long>> = {
              static _S_alignment = 8,
              _M_i = 1
            }, <No data fields>}
        },
        _mOwningThread = 1360,
        mStorageStream = [(nsStorageStream *) 0x7f2df3e24e20],
        mReadCursor = 0,
        mSegmentEnd = 0,
        mSegmentNum = 0,
        mSegmentSize = 4096,
        mLogicalCursor = 0,
        mStatus = nsresult::NS_OK
      }
    mLoadFlags:
      No mapping or pretty-printer for uint32_t, switching to gdb print.
      6881280
    mLoadGroup:
      mozilla::net::nsLoadGroup 0x7f2dea1f2690
        mLoadFlags:
          No mapping or pretty-printer for uint32_t, switching to gdb print.
          0
    mLoadInfo:
      mozilla::net::LoadInfo 0x7f2dea647de0
        mLoadingPrincipal:
          ContentPrincipal 0x7f2df683ef60
            mCodebase:
              mozilla::net::nsStandardURL 0x7f2e0623b100
                mSpec:
                  "https://www.youtube.com/edit?video_id=Xg3gfm3Fu14&video_referrer=watch"
            mOriginAttributes:
              mPrivateBrowsingId: 0 mUserContextId: 0 mFirstPartyDomain: <gNullChar> u""
        mOriginAttributes:
          mPrivateBrowsingId: 0 mUserContextId: 0 mFirstPartyDomain: <gNullChar> u""
Command name abbreviations are allowed if unambiguous.
(rr) cbt paste
  0 net::HttpChannelChild::ContinueAsyncOpen
    netwerk/protocol/http/HttpChannelChild.cpp:2506
  1 net::HttpChannelChild::ResetInterception
    netwerk/protocol/http/HttpChannelChild.cpp:3332
  2 net::InterceptedChannelContent::ResetInterception
    netwerk/protocol/http/InterceptedChannel.cpp:467
  3 dom::workers::(anonymous namespace)::ContinueDispatchFetchEventRunnable::HandleError
    dom/workers/ServiceWorkerManager.cpp:2664
  4 dom::workers::(anonymous namespace)::ContinueDispatchFetchEventRunnable::Run
    dom/workers/ServiceWorkerManager.cpp:2687
  5 nsPermissionManager::WhenPermissionsAvailable
    extensions/cookie/nsPermissionManager.cpp:3379
  6 dom::workers::ServiceWorkerManager::<lambda()>::operator()
    dom/workers/ServiceWorkerManager.cpp:2793
  7 detail::RunnableFunction<mozilla::dom::workers::ServiceWorkerManager::DispatchFetchEvent(const mozilla::OriginAttributes&, nsIDocument*, const nsAString&, nsIInterceptedChannel*, bool, bool, mozilla::ErrorResult&)::<lambda()> >::Run(void)
    obj-firefox-debug/dist/include/nsThreadUtils.h:507
  8 net::HttpBaseChannel::EnsureUploadStreamIsCloneableComplete
    netwerk/protocol/http/HttpBaseChannel.cpp:939
  9 detail::RunnableMethodArguments<nsresult>::applyImpl<mozilla::net::HttpChannelChild, void (mozilla::net::HttpBaseChannel::*)(nsresult), StoreCopyPassByConstLRef<nsresult>, 0ul>
    obj-firefox-debug/dist/include/nsThreadUtils.h:1122
 10 detail::RunnableMethodArguments<nsresult>::apply<mozilla::net::HttpChannelChild, void (mozilla::net::HttpBaseChannel::*)(nsresult)>
    obj-firefox-debug/dist/include/nsThreadUtils.h:1129
 11 detail::RunnableMethodImpl<mozilla::net::HttpChannelChild*, void (mozilla::net::HttpBaseChannel::*)(nsresult), true, (mozilla::RunnableKind)0, nsresult>::Run
    obj-firefox-debug/dist/include/nsThreadUtils.h:1172
 12 nsThread::ProcessNextEvent
    xpcom/threads/nsThread.cpp:1446
 13 NS_ProcessNextEvent
    xpcom/threads/nsThreadUtils.cpp:480
 14 ipc::MessagePump::Run
    ipc/glue/MessagePump.cpp:97
 15 ipc::MessagePumpForChildProcess::Run
    ipc/glue/MessagePump.cpp:302
 16 MessageLoop::RunInternal
    ipc/chromium/src/base/message_loop.cc:326
 17 MessageLoop::RunHandler
    ipc/chromium/src/base/message_loop.cc:319
 18 MessageLoop::Run
    ipc/chromium/src/base/message_loop.cc:299
 19 nsBaseAppShell::Run
    widget/nsBaseAppShell.cpp:156
 20 XRE_RunAppShell
    toolkit/xre/nsEmbedFunctions.cpp:882
 21 ipc::MessagePumpForChildProcess::Run
    ipc/glue/MessagePump.cpp:270
 22 MessageLoop::RunInternal
    ipc/chromium/src/base/message_loop.cc:326
 23 MessageLoop::RunHandler
    ipc/chromium/src/base/message_loop.cc:319
 24 MessageLoop::Run
    ipc/chromium/src/base/message_loop.cc:299
 25 XRE_InitChildProcess
    toolkit/xre/nsEmbedFunctions.cpp:699
 26 BootstrapImpl::XRE_InitChildProcess
    toolkit/xre/Bootstrap.cpp:65
 27 content_process_main
    browser/app/../../ipc/contentproc/plugin-container.cpp:64
 28 main
    browser/app/nsBrowserApp.cpp:285

```

investigating the upload stream a little bit...



### moot: Looking into suspcious SW errors

++DOMWINDOW == 14 (0x7f2de9981800) [pid = 1360] [serial = 14] [outer = 0x7f2de9a3c800]
[Child 1360] WARNING: 'NS_FAILED(rv) || NS_FAILED(status)', file /home/visbrero/rev_control/hg/mozilla-unified/dom/workers/ServiceWorkerManager.cpp, line 2686
[Child 1360] WARNING: Unexpected error while dispatching fetch event!: file /home/visbrero/rev_control/hg/mozilla-unified/dom/workers/ServiceWorkerManager.cpp, line 2663

This turns out just be session restore freaking out the service worker.

```
Thread 1 hit Breakpoint 1, mozilla::dom::workers::(anonymous namespace)::ContinueDispatchFetchEventRunnable::Run (this=0x7f2df6842d00) at /home/visbrero/rev_control/hg/mozilla-unified/dom/workers/ServiceWorkerManager.cpp:2686
2686	    if (NS_WARN_IF(NS_FAILED(rv) || NS_FAILED(status))) {
(rr) bt
#0  0x00007f2e133ee43c in mozilla::dom::workers::(anonymous namespace)::ContinueDispatchFetchEventRunnable::Run() (this=0x7f2df6842d00) at /home/visbrero/rev_control/hg/mozilla-unified/dom/workers/ServiceWorkerManager.cpp:2686
#1  0x00007f2e114bd3fa in nsPermissionManager::WhenPermissionsAvailable(nsIPrincipal*, nsIRunnable*) (this=<optimized out>, aPrincipal=<optimized out>, aRunnable=0x7f2df6842d00) at /home/visbrero/rev_control/hg/mozilla-unified/extensions/cookie/nsPermissionManager.cpp:3379
#2  0x00007f2e13405563 in mozilla::dom::workers::ServiceWorkerManager::<lambda()>::operator() (__closure=0x7f2df6925428) at /home/visbrero/rev_control/hg/mozilla-unified/dom/workers/ServiceWorkerManager.cpp:2793
#3  0x00007f2e13405563 in mozilla::detail::RunnableFunction<mozilla::dom::workers::ServiceWorkerManager::DispatchFetchEvent(const mozilla::OriginAttributes&, nsIDocument*, const nsAString&, nsIInterceptedChannel*, bool, bool, mozilla::ErrorResult&)::<lambda()> >::Run(void) (this=0x7f2df6925400) at /home/visbrero/rev_control/hg/mozilla-unified/obj-firefox-debug/dist/include/nsThreadUtils.h:507
#4  0x00007f2e10ab5fb0 in mozilla::net::HttpBaseChannel::EnsureUploadStreamIsCloneable(nsIRunnable*) (this=0x7f2df678e828, aCallback=0x7f2df6925400) at /home/visbrero/rev_control/hg/mozilla-unified/netwerk/protocol/http/HttpBaseChannel.cpp:865
#5  0x00007f2e13422e25 in mozilla::dom::workers::ServiceWorkerManager::DispatchFetchEvent(mozilla::OriginAttributes const&, nsIDocument*, nsAString const&, nsIInterceptedChannel*, bool, bool, mozilla::ErrorResult&) (this=<optimized out>, aOriginAttributes=..., aDoc=<optimized out>, aDocumentIdForTopLevelNavigation=..., aChannel=aChannel@entry=0x7f2df6779740, aIsReload=<optimized out>, aIsSubresourceLoad=false, aRv=...) at /home/visbrero/rev_control/hg/mozilla-unified/dom/workers/ServiceWorkerManager.cpp:2812
#6  0x00007f2e14dd0723 in nsDocShell::ChannelIntercepted(nsIInterceptedChannel*) (this=0x7f2e061cd800, aChannel=0x7f2df6779740) at /home/visbrero/rev_control/hg/mozilla-unified/docshell/base/nsDocShell.cpp:14946
#7  0x00007f2e10adebb9 in mozilla::net::InterceptedChannelBase::DoNotifyController() (this=this@entry=0x7f2df6779740) at /home/visbrero/rev_control/hg/mozilla-unified/netwerk/protocol/http/InterceptedChannel.cpp:78
#8  0x00007f2e10af9bbc in mozilla::net::InterceptedChannelContent::NotifyController() (this=0x7f2df6779740) at /home/visbrero/rev_control/hg/mozilla-unified/netwerk/protocol/http/InterceptedChannel.cpp:444
#9  0x00007f2e10b130e3 in mozilla::net::HttpChannelChild::AsyncOpen(nsIStreamListener*, nsISupports*) (this=this@entry=0x7f2df678e800, listener=<optimized out>, aContext=aContext@entry=0x0) at /home/visbrero/rev_control/hg/mozilla-unified/netwerk/protocol/http/HttpChannelChild.cpp:2372
#10 0x00007f2e10b13211 in mozilla::net::HttpChannelChild::AsyncOpen2(nsIStreamListener*) (this=0x7f2df678e800, aListener=<optimized out>) at /home/visbrero/rev_control/hg/mozilla-unified/netwerk/protocol/http/HttpChannelChild.cpp:2388
#11 0x00007f2e1154e592 in nsURILoader::OpenURI(nsIChannel*, unsigned int, nsIInterfaceRequestor*) (this=<optimized out>, channel=0x7f2df678e880, aFlags=<optimized out>, aWindowContext=0x7f2e061cd830) at /home/visbrero/rev_control/hg/mozilla-unified/uriloader/base/nsURILoader.cpp:857
#12 0x00007f2e14d917ec in nsDocShell::DoChannelLoad(nsIChannel*, nsIURILoader*, bool) (this=this@entry=0x7f2e061cd800, aChannel=0x7f2df678e880, aURILoader=0x7f2df76e49a0, aBypassClassifier=aBypassClassifier@entry=false) at /home/visbrero/rev_control/hg/mozilla-unified/docshell/base/nsDocShell.cpp:11594
#13 0x00007f2e14db0370 in nsDocShell::DoURILoad(nsIURI*, nsIURI*, mozilla::Maybe<nsCOMPtr<nsIURI> > const&, bool, bool, nsIURI*, bool, unsigned int, nsIPrincipal*, nsIPrincipal*, char const*, nsAString const&, nsIInputStream*, nsIInputStream*, bool, nsIDocShell**, nsIRequest**, bool, bool, bool, nsAString const&, nsIURI*, unsigned int) (this=this@entry=0x7f2e061cd800, aURI=aURI@entry=0x7f2e0623a4a0, aOriginalURI=aOriginalURI@entry=0x7f2e0623a6e0, aResultPrincipalURI=..., aLoadReplace=aLoadReplace@entry=false, aLoadFromExternal=aLoadFromExternal@entry=false, aReferrerURI=0x7f2e0623a5c0, aSendReferrer=true, aReferrerPolicy=4, aTriggeringPrincipal=0x7f2df684d060, aPrincipalToInherit=0x7f2df684d100, aTypeHint=0x7f2e170f5578 <gNullChar> "", aFileName=..., aPostData=0x0, aHeadersData=0x0, aFirstParty=true, aDocShell=0x0, aRequest=0x7ffdab267110, aIsNewWindowTarget=false, aBypassClassifier=false, aForceAllowCookies=false, aSrcdoc=..., aBaseURI=0x0, aContentPolicyType=6) at /home/visbrero/rev_control/hg/mozilla-unified/docshell/base/nsDocShell.cpp:11402
#14 0x00007f2e14dc1c8f in nsDocShell::InternalLoad(nsIURI*, nsIURI*, mozilla::Maybe<nsCOMPtr<nsIURI> > const&, bool, nsIURI*, unsigned int, nsIPrincipal*, nsIPrincipal*, unsigned int, nsAString const&, char const*, nsAString const&, nsIInputStream*, nsIInputStream*, unsigned int, nsISHEntry*, bool, nsAString const&, nsIDocShell*, nsIURI*, bool, nsIDocShell**, nsIRequest**) (this=this@entry=0x7f2e061cd800, aURI=<optimized out>, aOriginalURI=<optimized out>, aResultPrincipalURI=..., aLoadReplace=<optimized out>, aReferrer=<optimized out>, aReferrerPolicy=4, aTriggeringPrincipal=0x7f2df684d060, aPrincipalToInherit=0x7f2df684d100, aFlags=0, aWindowTarget=..., aTypeHint=0x7f2e170f5578 <gNullChar> "", aFileName=..., aPostData=0x0, aHeadersData=0x0, aLoadType=<optimized out>, aSHEntry=0x7f2df6779680, aFirstParty=true, aSrcdoc=..., aSourceDocShell=0x0, aBaseURI=0x0, aCheckForPrerender=false, aDocShell=0x0, aRequest=0x0) at /home/visbrero/rev_control/hg/mozilla-unified/docshell/base/nsDocShell.cpp:10788
#15 0x00007f2e14dc6dc2 in nsDocShell::LoadHistoryEntry(nsISHEntry*, unsigned int) (this=this@entry=0x7f2e061cd800, aEntry=0x7f2df6779680, aLoadType=aLoadType@entry=4) at /home/visbrero/rev_control/hg/mozilla-unified/docshell/base/nsDocShell.cpp:12773
#16 0x00007f2e14dc78a2 in nsDocShell::LoadURI(nsIURI*, nsIDocShellLoadInfo*, unsigned int, bool) (this=0x7f2e061cd800, aURI=0x7f2e0623a4a0, aLoadInfo=<optimized out>, aLoadFlags=0, aFirstParty=<optimized out>) at /home/visbrero/rev_control/hg/mozilla-unified/docshell/base/nsDocShell.cpp:1471
#17 0x00007f2e14debc26 in nsSHistory::InitiateLoad(nsISHEntry*, nsIDocShell*, long) (this=this@entry=0x7f2e0c10abc0, aFrameEntry=0x7f2df6779680, aFrameDS=0x7f2e061cd9a8, aLoadType=aLoadType@entry=2) at /home/visbrero/rev_control/hg/mozilla-unified/docshell/shistory/nsSHistory.cpp:1944
#18 0x00007f2e14decbd7 in nsSHistory::LoadEntry(int, long, unsigned int) (this=this@entry=0x7f2e0c10abc0, aIndex=7, aLoadType=aLoadType@entry=2, aHistCmd=aHistCmd@entry=3) at /home/visbrero/rev_control/hg/mozilla-unified/docshell/shistory/nsSHistory.cpp:1801
#19 0x00007f2e14ded175 in nsSHistory::ReloadCurrentEntry() (this=0x7f2e0c10abc0) at /home/visbrero/rev_control/hg/mozilla-unified/docshell/shistory/nsSHistory.cpp:1051
```

```
(rr) call DumpJSStack()
0 restoreTabContent(loadArguments = undefined, isRemotenessUpdate = false, finishCallback = () => {
      // Tell SessionStore.jsm that it may want to restore some more tabs,
      // since it restores a max of MAX_CONCURRENT_TAB_RESTORES at a time.
      sendAsyncMessage("SessionStore:restoreTabContentComplete", {epoch, isRemotenessUpdate});
    }) ["resource:///modules/sessionstore/ContentRestore.jsm":236]
    this = [object Object]
1 restoreTabContent(undefined, false, () => {
      // Tell SessionStore.jsm that it may want to restore some more tabs,
      // since it restores a max of MAX_CONCURRENT_TAB_RESTORES at a time.
      sendAsyncMessage("SessionStore:restoreTabContentComplete", {epoch, isRemotenessUpdate});
    }) ["self-hosted":956]
    this = [object Object]
2 restoreTabContent((destructured parameter) = [object Object]) ["chrome://browser/content/content-sessionStore.js":219]
    this = [object Object]
3 receiveMessage((destructured parameter) = [object Object]) ["chrome://browser/content/content-sessionStore.js":153]
    this = [object Object]
```


## Patch Analysis ##

Probably not that patch.

### Enclosing Case ###

https://hg.mozilla.org/mozilla-central/file/cf6065f64f9a/dom/file/ipc/Blob.cpp#l2772

This means the remote blob came from a different thread but we have a
BackgroundChild so we avoid the third case that does a blocking dispatch.

### Changed Logic ###

A new clause is added that, subsequent to getting the stream of the underlying
blob, checks if the remote blob is sliced, and if so, wraps the returned stream
in a SlicedInputStream using the CreateStreamHelper's mStart/mLength.

The helper's mStart saves off the RemoteBlobSliceImpl's provided aStart after
making it relative to the parent's own start if it was a slice.
