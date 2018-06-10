This file includes notes from various shutdown investigations.  Surprise!

## rr non-e10s shutdown
info breakpoints
```
Num     Type           Disp Enb Address            What
1       breakpoint     keep y   0x00007f52d5bc1330 in nsGlobalWindowInner::FreeInnerObjects() at /home/visbrero/rev_control/hg/mozilla-central/dom/base/nsGlobalWindowInner.cpp:1150
	breakpoint already hit 13 times
2       breakpoint     keep y   0x00007f52d92efb5a in nsAppStartup::CloseAllWindows() at /home/visbrero/rev_control/hg/mozilla-central/toolkit/components/startup/nsAppStartup.cpp:500
3       breakpoint     keep y   0x00007f52d92efeec in nsAppStartup::Quit(unsigned int) at /home/visbrero/rev_control/hg/mozilla-central/toolkit/components/startup/nsAppStartup.cpp:309
	breakpoint already hit 1 time
```

at first FreeInner after nsAppStartup::Quit (and using normal bt because cbt is going down some kind of symbol grokking rabbit hole):
```
(rr) bt
#0  0x00007f52d5bc1330 in nsGlobalWindowInner::FreeInnerObjects() (this=0x7f52aea0c000) at /home/visbrero/rev_control/hg/mozilla-central/dom/base/nsGlobalWindowInner.cpp:1150
#1  0x00007f52d5be703d in nsGlobalWindowOuter::DetachFromDocShell() (this=0x7f52aea08400) at /home/visbrero/rev_control/hg/mozilla-central/dom/base/nsGlobalWindowOuter.cpp:2177
#2  0x00007f52d8f54051 in nsDocShell::Destroy() (this=0x7f52ae888800) at /home/visbrero/rev_control/hg/mozilla-central/docshell/base/nsDocShell.cpp:5474
#3  0x00007f52d5d3b977 in nsFrameLoader::DestroyDocShell() (this=0x7f52bdc3b400) at /home/visbrero/rev_control/hg/mozilla-central/dom/base/nsFrameLoader.cpp:1839
#4  0x00007f52d5d3bab4 in nsFrameLoaderDestroyRunnable::Run() (this=0x7f5297585ec0) at /home/visbrero/rev_control/hg/mozilla-central/dom/base/nsFrameLoader.cpp:1777
#5  0x00007f52d5d953ad in nsIDocument::MaybeInitializeFinalizeFrameLoaders() (this=this@entry=0x7f52b4cb1000) at /home/visbrero/rev_control/hg/mozilla-central/dom/base/nsDocument.cpp:6730
#6  0x00007f52d5d955c5 in nsDocument::EndUpdate(unsigned int) (this=this@entry=0x7f52b4cb1000, aUpdateType=1) at /home/visbrero/rev_control/hg/mozilla-central/dom/base/nsDocument.cpp:5111
#7  0x00007f52d781940f in mozilla::dom::XULDocument::EndUpdate(unsigned int) (this=0x7f52b4cb1000, aUpdateType=<optimized out>) at /home/visbrero/rev_control/hg/mozilla-central/dom/xul/XULDocument.cpp:2954
#8  0x00007f52d5c545b7 in mozAutoDocUpdate::~mozAutoDocUpdate() (this=0x7ffc612f0d00, __in_chrg=<optimized out>) at /home/visbrero/rev_control/hg/mozilla-central/dom/base/mozAutoDocUpdate.h:40
#9  0x00007f52d5d4d18b in nsINode::doRemoveChildAt(unsigned int, bool, nsIContent*, nsAttrAndChildArray&) (this=this@entry=0x7f52c271ae50, aIndex=aIndex@entry=0, aNotify=aNotify@entry=true, aKid=0x7f52bdc359d0, aChildArray=...) at /home/visbrero/rev_control/hg/mozilla-central/dom/base/nsINode.cpp:1650
#10 0x00007f52d5c65fd8 in mozilla::dom::FragmentOrElement::RemoveChildAt_Deprecated(unsigned int, bool) (this=0x7f52c271ae50, aIndex=0, aNotify=<optimized out>) at /home/visbrero/rev_control/hg/mozilla-central/dom/base/FragmentOrElement.cpp:1211
#11 0x00007f52d5d341f7 in nsINode::RemoveChild(nsINode&, mozilla::ErrorResult&) (this=this@entry=0x7f52c271ae50, aOldChild=..., aError=...) at /home/visbrero/rev_control/hg/mozilla-central/dom/base/nsINode.cpp:545
#12 0x00007f52d5d35237 in nsINode::Remove() (this=this@entry=0x7f52bdc359d0) at /home/visbrero/rev_control/hg/mozilla-central/dom/base/nsINode.cpp:1577
#13 0x00007f52d6785c00 in mozilla::dom::ElementBinding::remove(JSContext*, JS::Handle<JSObject*>, mozilla::dom::Element*, JSJitMethodCallArgs const&) (cx=<optimized out>, obj=(JSObject * const) 0x7f52ae77f580 [object XULElement], self=0x7f52bdc359d0, args=...) at /home/visbrero/rev_control/hg/mozilla-central/obj-firefox-debug/dom/bindings/ElementBinding.cpp:4557
#14 0x00007f52d6a15fdd in mozilla::dom::binding_detail::GenericMethod<mozilla::dom::binding_detail::NormalThisPolicy, mozilla::dom::binding_detail::ThrowExceptions>(JSContext*, unsigned int, JS::Value*) (cx=<optimized out>, argc=<optimized out>, vp=<optimized out>) at /home/visbrero/rev_control/hg/mozilla-central/dom/bindings/BindingUtils.cpp:3260
#15 0x00007f52d957cbac in js::CallJSNative(JSContext*, bool (*)(JSContext*, unsigned int, JS::Value*), JS::CallArgs const&) (cx=cx@entry=0x7f52cf923000, native=native@entry=0x7f52d6a15eaa <mozilla::dom::binding_detail::GenericMethod<mozilla::dom::binding_detail::NormalThisPolicy, mozilla::dom::binding_detail::ThrowExceptions>(JSContext*, unsigned int, JS::Value*)>, args=...) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/JSContext-inl.h:280
#16 0x00007f52d956e0c5 in js::InternalCallOrConstruct(JSContext*, JS::CallArgs const&, js::MaybeConstruct) (cx=cx@entry=0x7f52cf923000, args=..., construct=construct@entry=js::NO_CONSTRUCT) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:467
#17 0x00007f52d956e47f in InternalCall(JSContext*, js::AnyInvokeArgs const&) (cx=cx@entry=0x7f52cf923000, args=...) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:516
#18 0x00007f52d956e5e9 in js::Call(JSContext*, JS::Handle<JS::Value>, JS::Handle<JS::Value>, js::AnyInvokeArgs const&, JS::MutableHandle<JS::Value>) (cx=cx@entry=0x7f52cf923000, fval=..., fval@entry=$JS::Value((JSObject *) 0x7f52ae7b7800 [object Function "remove"]), thisv=..., args=..., rval=$JS::Value((JSObject *) 0x7f52ae7b7800 [object Function "remove"])) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:535
#19 0x00007f52d9ab64a3 in js::ForwardingProxyHandler::call(JSContext*, JS::Handle<JSObject*>, JS::CallArgs const&) const (this=<optimized out>, cx=0x7f52cf923000, proxy=..., args=...) at /home/visbrero/rev_control/hg/mozilla-central/js/src/proxy/Wrapper.cpp:176
#20 0x00007f52d9aa5948 in js::CrossCompartmentWrapper::call(JSContext*, JS::Handle<JSObject*>, JS::CallArgs const&) const (this=0x7f52ddbf9680 <js::CrossCompartmentWrapper::singleton>, cx=<optimized out>, wrapper=..., args=...) at /home/visbrero/rev_control/hg/mozilla-central/js/src/proxy/CrossCompartmentWrapper.cpp:358
#21 0x00007f52d9a96043 in js::Proxy::call(JSContext*, JS::Handle<JSObject*>, JS::CallArgs const&) (cx=0x7f52cf923000, proxy=proxy@entry=(JSObject * const) 0x7f52be4ca040 [object Proxy], args=...) at /home/visbrero/rev_control/hg/mozilla-central/js/src/proxy/Proxy.cpp:510
#22 0x00007f52d9a960fd in js::proxy_Call(JSContext*, unsigned int, JS::Value*) (cx=<optimized out>, argc=<optimized out>, vp=<optimized out>) at /home/visbrero/rev_control/hg/mozilla-central/js/src/proxy/Proxy.cpp:769
#23 0x00007f52d957cbac in js::CallJSNative(JSContext*, bool (*)(JSContext*, unsigned int, JS::Value*), JS::CallArgs const&) (cx=cx@entry=0x7f52cf923000, native=0x7f52d9a9607b <js::proxy_Call(JSContext*, unsigned int, JS::Value*)>, args=...) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/JSContext-inl.h:280
#24 0x00007f52d956e233 in js::InternalCallOrConstruct(JSContext*, JS::CallArgs const&, js::MaybeConstruct) (cx=cx@entry=0x7f52cf923000, args=..., construct=construct@entry=js::NO_CONSTRUCT) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:449
#25 0x00007f52d956e47f in InternalCall(JSContext*, js::AnyInvokeArgs const&) (cx=cx@entry=0x7f52cf923000, args=...) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:516
#26 0x00007f52d956e5b5 in js::CallFromStack(JSContext*, JS::CallArgs const&) (cx=cx@entry=0x7f52cf923000, args=...) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:522
#27 0x00007f52d9567c01 in Interpret(JSContext*, js::RunState&) (cx=cx@entry=0x7f52cf923000, state=...) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:3086
#28 0x00007f52d956d89d in js::RunScript(JSContext*, js::RunState&) (cx=cx@entry=0x7f52cf923000, state=...) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:417
#29 0x00007f52d956e3b4 in js::InternalCallOrConstruct(JSContext*, JS::CallArgs const&, js::MaybeConstruct) (cx=cx@entry=0x7f52cf923000, args=..., construct=construct@entry=js::NO_CONSTRUCT) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:489
#30 0x00007f52d956e47f in InternalCall(JSContext*, js::AnyInvokeArgs const&) (cx=cx@entry=0x7f52cf923000, args=...) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:516
#31 0x00007f52d956e5e9 in js::Call(JSContext*, JS::Handle<JS::Value>, JS::Handle<JS::Value>, js::AnyInvokeArgs const&, JS::MutableHandle<JS::Value>) (cx=cx@entry=0x7f52cf923000, fval=..., fval@entry=$JS::Value((JSObject *) 0x7f52be482560 [object Function "shutdown"]), thisv=..., args=..., rval=$JS::Value((JSObject *) 0x7f52be482560 [object Function "shutdown"])) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:535
#32 0x00007f52d9ab64a3 in js::ForwardingProxyHandler::call(JSContext*, JS::Handle<JSObject*>, JS::CallArgs const&) const (this=<optimized out>, cx=0x7f52cf923000, proxy=..., args=...) at /home/visbrero/rev_control/hg/mozilla-central/js/src/proxy/Wrapper.cpp:176
#33 0x00007f52d9aa5948 in js::CrossCompartmentWrapper::call(JSContext*, JS::Handle<JSObject*>, JS::CallArgs const&) const (this=0x7f52ddbf9680 <js::CrossCompartmentWrapper::singleton>, cx=<optimized out>, wrapper=..., args=...) at /home/visbrero/rev_control/hg/mozilla-central/js/src/proxy/CrossCompartmentWrapper.cpp:358
#34 0x00007f52d9a96043 in js::Proxy::call(JSContext*, JS::Handle<JSObject*>, JS::CallArgs const&) (cx=0x7f52cf923000, proxy=proxy@entry=(JSObject * const) 0x7f52be4808c0 [object Proxy], args=...) at /home/visbrero/rev_control/hg/mozilla-central/js/src/proxy/Proxy.cpp:510
#35 0x00007f52d9a960fd in js::proxy_Call(JSContext*, unsigned int, JS::Value*) (cx=<optimized out>, argc=<optimized out>, vp=<optimized out>) at /home/visbrero/rev_control/hg/mozilla-central/js/src/proxy/Proxy.cpp:769
#36 0x00007f52d957cbac in js::CallJSNative(JSContext*, bool (*)(JSContext*, unsigned int, JS::Value*), JS::CallArgs const&) (cx=cx@entry=0x7f52cf923000, native=0x7f52d9a9607b <js::proxy_Call(JSContext*, unsigned int, JS::Value*)>, args=...) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/JSContext-inl.h:280
#37 0x00007f52d956e233 in js::InternalCallOrConstruct(JSContext*, JS::CallArgs const&, js::MaybeConstruct) (cx=cx@entry=0x7f52cf923000, args=..., construct=construct@entry=js::NO_CONSTRUCT) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:449
#38 0x00007f52d956e47f in InternalCall(JSContext*, js::AnyInvokeArgs const&) (cx=cx@entry=0x7f52cf923000, args=...) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:516
#39 0x00007f52d956e5b5 in js::CallFromStack(JSContext*, JS::CallArgs const&) (cx=cx@entry=0x7f52cf923000, args=...) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:522
#40 0x00007f52d9567c01 in Interpret(JSContext*, js::RunState&) (cx=cx@entry=0x7f52cf923000, state=...) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:3086
#41 0x00007f52d956d89d in js::RunScript(JSContext*, js::RunState&) (cx=cx@entry=0x7f52cf923000, state=...) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:417
#42 0x00007f52d956e3b4 in js::InternalCallOrConstruct(JSContext*, JS::CallArgs const&, js::MaybeConstruct) (cx=cx@entry=0x7f52cf923000, args=..., construct=construct@entry=js::NO_CONSTRUCT) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:489
#43 0x00007f52d956e47f in InternalCall(JSContext*, js::AnyInvokeArgs const&) (cx=cx@entry=0x7f52cf923000, args=...) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:516
#44 0x00007f52d956e5e9 in js::Call(JSContext*, JS::Handle<JS::Value>, JS::Handle<JS::Value>, js::AnyInvokeArgs const&, JS::MutableHandle<JS::Value>) (cx=cx@entry=0x7f52cf923000, fval=..., fval@entry=$JS::Value((JSObject *) 0x7f52a7e341f0 [object Function "onShutdown"]), thisv=..., args=..., rval=$JS::Value((JSObject *) 0x7f52a7e341f0 [object Function "onShutdown"])) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:535
#45 0x00007f52d9ab64a3 in js::ForwardingProxyHandler::call(JSContext*, JS::Handle<JSObject*>, JS::CallArgs const&) const (this=<optimized out>, cx=0x7f52cf923000, proxy=..., args=...) at /home/visbrero/rev_control/hg/mozilla-central/js/src/proxy/Wrapper.cpp:176
#46 0x00007f52d9aa5948 in js::CrossCompartmentWrapper::call(JSContext*, JS::Handle<JSObject*>, JS::CallArgs const&) const (this=0x7f52ddbf9680 <js::CrossCompartmentWrapper::singleton>, cx=<optimized out>, wrapper=..., args=...) at /home/visbrero/rev_control/hg/mozilla-central/js/src/proxy/CrossCompartmentWrapper.cpp:358
#47 0x00007f52d9a96043 in js::Proxy::call(JSContext*, JS::Handle<JSObject*>, JS::CallArgs const&) (cx=0x7f52cf923000, proxy=proxy@entry=(JSObject * const) 0x7f52be480040 [object Proxy], args=...) at /home/visbrero/rev_control/hg/mozilla-central/js/src/proxy/Proxy.cpp:510
#48 0x00007f52d9a960fd in js::proxy_Call(JSContext*, unsigned int, JS::Value*) (cx=<optimized out>, argc=<optimized out>, vp=<optimized out>) at /home/visbrero/rev_control/hg/mozilla-central/js/src/proxy/Proxy.cpp:769
#49 0x00007f52d957cbac in js::CallJSNative(JSContext*, bool (*)(JSContext*, unsigned int, JS::Value*), JS::CallArgs const&) (cx=cx@entry=0x7f52cf923000, native=0x7f52d9a9607b <js::proxy_Call(JSContext*, unsigned int, JS::Value*)>, args=...) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/JSContext-inl.h:280
#50 0x00007f52d956e233 in js::InternalCallOrConstruct(JSContext*, JS::CallArgs const&, js::MaybeConstruct) (cx=cx@entry=0x7f52cf923000, args=..., construct=construct@entry=js::NO_CONSTRUCT) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:449
#51 0x00007f52d956e47f in InternalCall(JSContext*, js::AnyInvokeArgs const&) (cx=cx@entry=0x7f52cf923000, args=...) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:516
#52 0x00007f52d956e5b5 in js::CallFromStack(JSContext*, JS::CallArgs const&) (cx=cx@entry=0x7f52cf923000, args=...) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:522
#53 0x00007f52d9567c01 in Interpret(JSContext*, js::RunState&) (cx=cx@entry=0x7f52cf923000, state=...) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:3086
#54 0x00007f52d956d89d in js::RunScript(JSContext*, js::RunState&) (cx=cx@entry=0x7f52cf923000, state=...) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:417
#55 0x00007f52d956e3b4 in js::InternalCallOrConstruct(JSContext*, JS::CallArgs const&, js::MaybeConstruct) (cx=cx@entry=0x7f52cf923000, args=..., construct=construct@entry=js::NO_CONSTRUCT) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:489
#56 0x00007f52d956e47f in InternalCall(JSContext*, js::AnyInvokeArgs const&) (cx=cx@entry=0x7f52cf923000, args=...) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:516
#57 0x00007f52d956e5e9 in js::Call(JSContext*, JS::Handle<JS::Value>, JS::Handle<JS::Value>, js::AnyInvokeArgs const&, JS::MutableHandle<JS::Value>) (cx=cx@entry=0x7f52cf923000, fval=..., fval@entry=$JS::Value((JSObject *) 0x7f52a7f911f0 [object Function "ExtensionAPI/<"]), thisv=..., thisv@entry=$JS::UndefinedValue(), args=..., rval=$JS::Value((JSObject *) 0x7f52a7f911f0 [object Function "ExtensionAPI/<"])) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:535
#58 0x00007f52d956ff91 in js::SpreadCallOperation(JSContext*, JS::Handle<JSScript*>, unsigned char*, JS::Handle<JS::Value>, JS::Handle<JS::Value>, JS::Handle<JS::Value>, JS::Handle<JS::Value>, JS::MutableHandle<JS::Value>) (cx=cx@entry=0x7f52cf923000, script=..., script@entry=0x7f52c8e71128, pc=0x7f52cbce8fdf ")\005\231\235\220\006ߘ\220\006ۘ\230\220\006\313\b͐\020\330\070!\361\230\220\004", thisv=..., thisv@entry=$JS::UndefinedValue(), callee=..., callee@entry=$JS::Value((JSObject *) 0x7f52a7f911f0 [object Function "ExtensionAPI/<"]), arr=..., arr@entry=$JS::Value((JSObject *) 0x7f52cc10e548 [object Array]), newTarget=$JS::NullValue(), res=$JS::Value((JSObject *) 0x7f52a7f911f0 [object Function "ExtensionAPI/<"])) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:4902
#59 0x00007f52d95675ee in Interpret(JSContext*, js::RunState&) (cx=cx@entry=0x7f52cf923000, state=...) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:3036
#60 0x00007f52d956d89d in js::RunScript(JSContext*, js::RunState&) (cx=cx@entry=0x7f52cf923000, state=...) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:417
#61 0x00007f52d956e3b4 in js::InternalCallOrConstruct(JSContext*, JS::CallArgs const&, js::MaybeConstruct) (cx=cx@entry=0x7f52cf923000, args=..., construct=construct@entry=js::NO_CONSTRUCT) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:489
#62 0x00007f52d956e47f in InternalCall(JSContext*, js::AnyInvokeArgs const&) (cx=cx@entry=0x7f52cf923000, args=...) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:516
#63 0x00007f52d956e5e9 in js::Call(JSContext*, JS::Handle<JS::Value>, JS::Handle<JS::Value>, js::AnyInvokeArgs const&, JS::MutableHandle<JS::Value>) (cx=cx@entry=0x7f52cf923000, fval=..., fval@entry=$JS::Value((JSObject *) 0x7f52a7f78f10 [object Function "wrapper"]), thisv=..., thisv@entry=$JS::UndefinedValue(), args=..., rval=$JS::Value((JSObject *) 0x7f52a7f78f10 [object Function "wrapper"])) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:535
#64 0x00007f52d956ff91 in js::SpreadCallOperation(JSContext*, JS::Handle<JSScript*>, unsigned char*, JS::Handle<JS::Value>, JS::Handle<JS::Value>, JS::Handle<JS::Value>, JS::Handle<JS::Value>, JS::MutableHandle<JS::Value>) (cx=cx@entry=0x7f52cf923000, script=..., script@entry=0x7f52c8e711c0, pc=0x7f52cbf56d6f ")\213\004", thisv=..., thisv@entry=$JS::UndefinedValue(), callee=..., callee@entry=$JS::Value((JSObject *) 0x7f52a7f78f10 [object Function "wrapper"]), arr=..., arr@entry=$JS::Value((JSObject *) 0x7f52cc10e498 [object Array]), newTarget=$JS::NullValue(), res=$JS::Value((JSObject *) 0x7f52a7f78f10 [object Function "wrapper"])) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:4902
#65 0x00007f52d95675ee in Interpret(JSContext*, js::RunState&) (cx=cx@entry=0x7f52cf923000, state=...) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:3036
#66 0x00007f52d956d89d in js::RunScript(JSContext*, js::RunState&) (cx=cx@entry=0x7f52cf923000, state=...) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:417
#67 0x00007f52d956e3b4 in js::InternalCallOrConstruct(JSContext*, JS::CallArgs const&, js::MaybeConstruct) (cx=cx@entry=0x7f52cf923000, args=..., construct=construct@entry=js::NO_CONSTRUCT) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:489
#68 0x00007f52d956e47f in InternalCall(JSContext*, js::AnyInvokeArgs const&) (cx=cx@entry=0x7f52cf923000, args=...) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:516
#69 0x00007f52d956e5e9 in js::Call(JSContext*, JS::Handle<JS::Value>, JS::Handle<JS::Value>, js::AnyInvokeArgs const&, JS::MutableHandle<JS::Value>) (cx=cx@entry=0x7f52cf923000, fval=..., fval@entry=$JS::Value((JSObject *) 0x7f52c6d96c40 [object Function "emit"]), thisv=..., thisv@entry=$JS::Value((JSObject *) 0x7f52b9f6a740 [object Object]), args=..., rval=$JS::Value((JSObject *) 0x7f52c6d96c40 [object Function "emit"])) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:535
#70 0x00007f52d956ff91 in js::SpreadCallOperation(JSContext*, JS::Handle<JSScript*>, unsigned char*, JS::Handle<JS::Value>, JS::Handle<JS::Value>, JS::Handle<JS::Value>, JS::Handle<JS::Value>, JS::MutableHandle<JS::Value>) (cx=cx@entry=0x7f52cf923000, script=..., script@entry=0x7f52c31fad08, pc=0x7f52c4508b34 ")\005\231ğϐ\b\v\236\220\006ߐ>Ȑ\aȐ\377\377\377\273\235\230\230\220\004Ր\031ɐ\a\330\070!\360\230\220\002", thisv=..., thisv@entry=$JS::Value((JSObject *) 0x7f52b9f6a740 [object Object]), callee=..., callee@entry=$JS::Value((JSObject *) 0x7f52c6d96c40 [object Function "emit"]), arr=..., arr@entry=$JS::Value((JSObject *) 0x7f52cc10e298 [object Array]), newTarget=$JS::NullValue(), res=$JS::Value((JSObject *) 0x7f52c6d96c40 [object Function "emit"])) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:4902
#71 0x00007f52d95675ee in Interpret(JSContext*, js::RunState&) (cx=cx@entry=0x7f52cf923000, state=...) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:3036
#72 0x00007f52d956d89d in js::RunScript(JSContext*, js::RunState&) (cx=cx@entry=0x7f52cf923000, state=...) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:417
#73 0x00007f52d956e3b4 in js::InternalCallOrConstruct(JSContext*, JS::CallArgs const&, js::MaybeConstruct) (cx=cx@entry=0x7f52cf923000, args=..., construct=construct@entry=js::NO_CONSTRUCT) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:489
#74 0x00007f52d956e47f in InternalCall(JSContext*, js::AnyInvokeArgs const&) (cx=cx@entry=0x7f52cf923000, args=...) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:516
#75 0x00007f52d956e5e9 in js::Call(JSContext*, JS::Handle<JS::Value>, JS::Handle<JS::Value>, js::AnyInvokeArgs const&, JS::MutableHandle<JS::Value>) (cx=cx@entry=0x7f52cf923000, fval=..., fval@entry=$JS::Value((JSObject *) 0x7f52c6d54560 [object Function "InterpretGeneratorResume"]), thisv=..., args=..., rval=$JS::UndefinedValue()) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Interpreter.cpp:535
#76 0x00007f52d9993afa in js::jit::InterpretResume(JSContext*, JS::Handle<JSObject*>, JS::Handle<JS::Value>, JS::Handle<js::PropertyName*>, JS::MutableHandle<JS::Value>) (cx=0x7f52cf923000, obj=..., val=$JS::UndefinedValue(), kind="next", rval=...) at /home/visbrero/rev_control/hg/mozilla-central/js/src/jit/VMFunctions.cpp:944
#77 0x00001bf09ec38e3f in  ()
#78 0x00007ffc612f4d70 in  ()
#79 0x00007ffc612f4d58 in  ()
#80 0x00007f52e9fb6740 in  ()
#81 0xfff9800000000000 in  ()
#82 0x00007f52ddecbc00 in InterpretResumeInfo () at /home/visbrero/rev_control/hg/mozilla-central/obj-firefox-debug/dist/bin/libxul.so
#83 0x00001bf09ecdbbfd in  ()
#84 0x0000000000007821 in  ()
#85 0x00007f52cc10c718 in  ()
#86 0xfff9800000000000 in  ()
#87 0x00007f52cc026560 in  ()
#88 0x00007ffc612f4de8 in  ()
#89 0xfff9800000000000 in  ()
#90 0xfffe7f52cc10c718 in  ()
#91 0xfff9800000000000 in  ()
#92 0xfffe7f52cc10c718 in  ()
#93 0x00007f52d9d2ece6 in js::jit::JitActivation::JitActivation(JSContext*) (this=0x7ffc612f4da8, cx=0x0) at /home/visbrero/rev_control/hg/mozilla-central/js/src/vm/Stack.cpp:1532
#94 0x00001bf09ec35ad7 in  ()
#95 0x00000000000020c3 in  ()
#96 0x00007f52cc0a5240 in  ()
#97 0x0000000000000001 in  ()
#98 0xfffe7f52cc10c718 in  ()
#99 0xfff9800000000000 in  ()
#100 0x00007ffc612f4ec0 in  ()
#101 0x0000000000000000 in  ()

(rr) call DumpJSStack()
0 shutdown() ["resource://gre/modules/ExtensionParent.jsm":1176]
    this = [object Object]
1 shutdown() ["chrome://extensions/content/parent/ext-backgroundPage.js":54]
    this = [object Object]
2 onShutdown("APP_SHUTDOWN") ["chrome://extensions/content/parent/ext-backgroundPage.js":93]
    this = [object Object]
3 ExtensionAPI/<("shutdown") ["resource://gre/modules/ExtensionCommon.jsm":101]
4 wrapper(args = shutdown) ["resource://gre/modules/ExtensionUtils.jsm":199]
5 emit(event = "shutdown") ["resource://gre/modules/ExtensionUtils.jsm":227]
    this = [object Object]
6 emit(event = "shutdown") ["resource://gre/modules/Extension.jsm":1355]
    this = [object Object]
7 _shutdown() ["resource://gre/modules/Extension.jsm":1822]
8 InterpretGeneratorResume(gen = [object Generator], val = undefined, kind = "next") ["self-hosted":1274]
9 next(val = undefined) ["self-hosted":1229]
    this = [object Generator]
10 shutdown() ["resource://gre/modules/Extension.jsm":1744]
11 InterpretGeneratorResume(gen = [object Generator], val = undefined, kind = "next") ["self-hosted":1274]
12 next(val = undefined) ["self-hosted":1229]
    this = [object Generator]
13 shutdown() ["resource://gre/modules/LegacyExtensionsUtils.jsm":215]
14 InterpretGeneratorResume(gen = [object Generator], val = undefined, kind = "next") ["self-hosted":1274]
15 next(val = undefined) ["self-hosted":1229]
    this = [object Generator]
16 callBootstrapMethod(aAddon = [object Object], aFile = [xpconnect wrapped nsIFile @ 0x7f52c5aebdc0 (native @ 0x7f52c5af3640)], aMethod = "shutdown", aReason = 2) ["resource://gre/modules/addons/XPIProvider.jsm":2741]
    this = [object Object]
17 observe(aSubject = null, aTopic = "quit-application-granted", aData = null) ["resource://gre/modules/addons/XPIProvider.jsm":1669]
    this = [object Object]
18 goQuitApplication() ["chrome://global/content/globalOverlay.js":56]
    this = [object ChromeWindow]
19 oncommand(event = [object XULCommandEvent]) ["chrome://browser/content/browser.xul":1]
    this = [object XULElement]
```

okay, that happens a bunch, then we get to CloseAllWindows()
```
(rr) cbt paste
  0 nsAppStartup::CloseAllWindows
    toolkit/components/startup/nsAppStartup.cpp:500
  1 nsAppStartup::Quit
    toolkit/components/startup/nsAppStartup.cpp:427
  2 NS_InvokeByIndex
    xpcom/reflect/xptcall/md/unix/xptcinvoke_asm_x86_64_unix.S:106
  3 CallMethodHelper::Invoke
    js/xpconnect/src/XPCWrappedNative.cpp:1962
  4 CallMethodHelper::Call
    js/xpconnect/src/XPCWrappedNative.cpp:1268
  5 XPCWrappedNative::CallMethod
    js/xpconnect/src/XPCWrappedNative.cpp:1235
  6 XPC_WN_CallMethod
    js/xpconnect/src/XPCWrappedNativeJSOps.cpp:911
  7 js::CallJSNative
    js/src/vm/JSContext-inl.h:280
  8 js::InternalCallOrConstruct
    js/src/vm/Interpreter.cpp:467
  9 InternalCall
    js/src/vm/Interpreter.cpp:516
 10 js::CallFromStack
    js/src/vm/Interpreter.cpp:522
 11 Interpret
    js/src/vm/Interpreter.cpp:3086
 12 js::RunScript
    js/src/vm/Interpreter.cpp:417
 13 js::InternalCallOrConstruct
    js/src/vm/Interpreter.cpp:489
 14 InternalCall
    js/src/vm/Interpreter.cpp:516
 15 js::Call
    js/src/vm/Interpreter.cpp:535
 16 JS::Call
    js/src/jsapi.cpp:2989
 17 dom::EventHandlerNonNull::Call
    obj-firefox-debug/dom/bindings/EventHandlerBinding.cpp:264
 18 dom::EventHandlerNonNull::Call<nsISupports*>
    obj-firefox-debug/dist/include/mozilla/dom/EventHandlerBinding.h:363
 19 JSEventHandler::HandleEvent
    dom/events/JSEventHandler.cpp:214
 20 EventListenerManager::HandleEventSubType
    dom/events/EventListenerManager.cpp:1121
 21 EventListenerManager::HandleEventInternal
    dom/events/EventListenerManager.cpp:1288
 22 EventListenerManager::HandleEvent
    obj-firefox-debug/dist/include/mozilla/EventListenerManager.h:392
 23 EventTargetChainItem::HandleEvent
    dom/events/EventDispatcher.cpp:348
 24 EventTargetChainItem::HandleEventTargetChain
    dom/events/EventDispatcher.cpp:528
 25 EventDispatcher::Dispatch
    dom/events/EventDispatcher.cpp:934
 26 EventDispatcher::DispatchDOMEvent
    dom/events/EventDispatcher.cpp:1014
 27 nsINode::DispatchEvent
    dom/base/nsINode.cpp:1079
 28 dom::EventTarget::DispatchEvent
    dom/events/EventTarget.cpp:212
 29 nsContentUtils::DispatchXULCommand
    dom/base/nsContentUtils.cpp:6534
 30 nsXULElement::DispatchXULCommand
    dom/xul/nsXULElement.cpp:1327
 31 nsXULElement::PreHandleEvent
    dom/xul/nsXULElement.cpp:1388
 32 EventTargetChainItem::PreHandleEvent
    dom/events/EventDispatcher.cpp:446
 33 EventDispatcher::Dispatch
    dom/events/EventDispatcher.cpp:860
 34 EventDispatcher::DispatchDOMEvent
    dom/events/EventDispatcher.cpp:1014
 35 PresShell::HandleDOMEventWithTarget
    layout/base/PresShell.cpp:8111
 36 nsContentUtils::DispatchXULCommand
    dom/base/nsContentUtils.cpp:6530
 37 nsButtonBoxFrame::DoMouseClick
    layout/xul/nsButtonBoxFrame.cpp:230
 38 nsButtonBoxFrame::MouseClicked
    layout/xul/nsButtonBoxFrame.h:36
 39 nsButtonBoxFrame::HandleEvent
    layout/xul/nsButtonBoxFrame.cpp:171
 40 nsPresShellEventCB::HandleEvent
    layout/base/PresShell.cpp:510
 41 EventTargetChainItem::HandleEventTargetChain
    dom/events/EventDispatcher.cpp:583
 42 EventDispatcher::Dispatch
    dom/events/EventDispatcher.cpp:934
 43 PresShell::DispatchEventToDOM
    layout/base/PresShell.cpp:7977
 44 PresShell::HandleEventInternal
    layout/base/PresShell.cpp:7654
 45 PresShell::HandleEventWithTarget
    layout/base/PresShell.cpp:7439
 46 EventStateManager::InitAndDispatchClickEvent
    dom/events/EventStateManager.cpp:5055
 47 EventStateManager::CheckForAndDispatchClick
    dom/events/EventStateManager.cpp:5097
 48 EventStateManager::PostHandleEvent
    dom/events/EventStateManager.cpp:3460
 49 PresShell::HandleEventInternal
    layout/base/PresShell.cpp:7674
 50 PresShell::HandleEvent
    layout/base/PresShell.cpp:7264
 51 nsViewManager::DispatchEvent
    view/nsViewManager.cpp:812
 52 nsView::HandleEvent
    view/nsView.cpp:1140
 53 nsWindow::DispatchEvent
    widget/gtk/nsWindow.cpp:578
 54 nsBaseWidget::DispatchInputEvent
    widget/nsBaseWidget.cpp:1238
 55 nsWindow::OnButtonReleaseEvent
    widget/gtk/nsWindow.cpp:2817
 56 button_release_event_cb
    widget/gtk/nsWindow.cpp:5747
 57 ??
    ???:0
 58 g_closure_invoke
    ???:0
 59 ??
    ???:0
 60 g_signal_emit_valist
    ???:0
 61 g_signal_emit
    ???:0
 62 ??
    ???:0
 63 ??
    ???:0
 64 gtk_main_do_event
    ???:0
 65 ??
    ???:0
 66 ??
    ???:0
 67 g_main_context_dispatch
    ???:0
 68 ??
    ???:0
 69 g_main_context_iteration
    ???:0
 70 nsAppShell::ProcessNextNativeEvent
    widget/gtk/nsAppShell.cpp:295
 71 nsBaseAppShell::DoProcessNextNativeEvent
    widget/nsBaseAppShell.cpp:139
 72 nsBaseAppShell::OnProcessNextEvent
    widget/nsBaseAppShell.cpp:290
 73 nsThread::ProcessNextEvent
    xpcom/threads/nsThread.cpp:979
 74 NS_ProcessNextEvent
    xpcom/threads/nsThreadUtils.cpp:519
 75 ipc::MessagePump::Run
    ipc/glue/MessagePump.cpp:125
 76 MessageLoop::RunInternal
    ipc/chromium/src/base/message_loop.cc:326
 77 MessageLoop::RunHandler
    ipc/chromium/src/base/message_loop.cc:319
 78 MessageLoop::Run
    ipc/chromium/src/base/message_loop.cc:299
 79 nsBaseAppShell::Run
    widget/nsBaseAppShell.cpp:157
 80 nsAppStartup::Run
    toolkit/components/startup/nsAppStartup.cpp:290
 81 XREMain::XRE_mainRun
    toolkit/xre/nsAppRunner.cpp:4829
 82 XREMain::XRE_main
    toolkit/xre/nsAppRunner.cpp:4974
 83 XRE_main
    toolkit/xre/nsAppRunner.cpp:5066
 84 BootstrapImpl::XRE_main
    toolkit/xre/Bootstrap.cpp:49
 85 do_main
    browser/app/nsBrowserApp.cpp:231
 86 main
    browser/app/nsBrowserApp.cpp:304

(rr) call DumpJSStack()
0 goQuitApplication() ["chrome://global/content/globalOverlay.js":56]
    this = [object ChromeWindow]
1 oncommand(event = [object XULCommandEvent]) ["chrome://browser/content/browser.xul":1]
    this = [object XULElement]
```

okay, then we get to an actual free:
```
Thread 1 hit Breakpoint 1, nsGlobalWindowInner::FreeInnerObjects (this=0x7f52bef50400) at /home/visbrero/rev_control/hg/mozilla-central/dom/base/nsGlobalWindowInner.cpp:1150
1150	{
(rr) cbt paste
  0 nsGlobalWindowInner::FreeInnerObjects
    dom/base/nsGlobalWindowInner.cpp:1150
  1 nsGlobalWindowOuter::DetachFromDocShell
    dom/base/nsGlobalWindowOuter.cpp:2177
  2 nsDocShell::Destroy
    docshell/base/nsDocShell.cpp:5474
  3 nsFrameLoader::DestroyDocShell
    dom/base/nsFrameLoader.cpp:1839
  4 nsFrameLoaderDestroyRunnable::Run
    dom/base/nsFrameLoader.cpp:1777
  5 nsIDocument::MaybeInitializeFinalizeFrameLoaders
    dom/base/nsDocument.cpp:6730
  6 detail::RunnableMethodArguments<>::applyImpl<nsIDocument, void (nsIDocument::*)()>(nsIDocument*, void (nsIDocument::*)(), mozilla::Tuple<>&, std::integer_sequence<unsigned long>)
    obj-firefox-debug/dist/include/nsThreadUtils.h:1165
  7 _ZN7mozilla6detail23RunnableMethodArgumentsIJEE5applyI11nsIDocumentMS4_FvvEEEDTcl9applyImplfp_fp0_dtdefpT10mArgumentstlSt16integer_sequenceImJEEEEEPT_T0_
    obj-firefox-debug/dist/include/nsThreadUtils.h:1171
  8 detail::RunnableMethodImpl<nsIDocument*, void (nsIDocument::*)(), true, (mozilla::RunnableKind)0>::Run
    obj-firefox-debug/dist/include/nsThreadUtils.h:1216
  9 nsContentUtils::RemoveScriptBlocker
    dom/base/nsContentUtils.cpp:5635
 10 nsAutoScriptBlocker::~nsAutoScriptBlocker
    dom/base/nsContentUtils.h:3547
 11 nsDocumentViewer::Destroy
    layout/base/nsDocumentViewer.cpp:1769
 12 nsDocShell::Destroy
    docshell/base/nsDocShell.cpp:5464
 13 nsXULWindow::Destroy
    xpfe/appshell/nsXULWindow.cpp:526
 14 nsWebShellWindow::Destroy
    xpfe/appshell/nsWebShellWindow.cpp:776
 15 nsChromeTreeOwner::Destroy
    xpfe/appshell/nsChromeTreeOwner.cpp:294
 16 nsGlobalWindowOuter::ReallyCloseWindow
    dom/base/nsGlobalWindowOuter.cpp:6035
 17 nsCloseEvent::Run
    dom/base/nsGlobalWindowOuter.cpp:5806
 18 nsThread::ProcessNextEvent
    xpcom/threads/nsThread.cpp:1090
 19 NS_ProcessNextEvent
    xpcom/threads/nsThreadUtils.cpp:519
 20 ipc::MessagePump::Run
    ipc/glue/MessagePump.cpp:97
 21 MessageLoop::RunInternal
    ipc/chromium/src/base/message_loop.cc:326
 22 MessageLoop::RunHandler
    ipc/chromium/src/base/message_loop.cc:319
 23 MessageLoop::Run
    ipc/chromium/src/base/message_loop.cc:299
 24 nsBaseAppShell::Run
    widget/nsBaseAppShell.cpp:157
 25 nsAppStartup::Run
    toolkit/components/startup/nsAppStartup.cpp:290
 26 XREMain::XRE_mainRun
    toolkit/xre/nsAppRunner.cpp:4829
 27 XREMain::XRE_main
    toolkit/xre/nsAppRunner.cpp:4974
 28 XRE_main
    toolkit/xre/nsAppRunner.cpp:5066
 29 BootstrapImpl::XRE_main
    toolkit/xre/Bootstrap.cpp:49
 30 do_main
    browser/app/nsBrowserApp.cpp:231
 31 main
    browser/app/nsBrowserApp.cpp:304
```
with doc of about:blank

next:
```
Thread 1 hit Breakpoint 1, nsGlobalWindowInner::FreeInnerObjects (this=0x7f52b6680800) at /home/visbrero/rev_control/hg/mozilla-central/dom/base/nsGlobalWindowInner.cpp:1150
1150	{
(rr) pp this
nsGlobalWindowInner 0x7f52b6680800
  doc:
    mDoc:
      nsHTMLDocument 0x7f529e2f8000
        URIs:
          mDocumentURI:
            mozilla::net::nsStandardURL 0x7f52bf2ee000
              mSpec:
                0x7f52bdd6218c "http://example.com/"
          mOriginalURI:
            mozilla::net::nsStandardURL 0x7f52bf2ee000
              mSpec:
                0x7f52bdd6218c "http://example.com/"
        state:
          mReadyState:
            nsIDocument::READYSTATE_COMPLETE
          mVisibilityState:
            mozilla::dom::VisibilityState::Hidden
          mOnloadBlockCount:
            1
          mBFCacheEntry:
            0x0
          mIsThirdParty:
            No mapping or pretty-printer for mozilla::Maybe, switching to gdb print.
            {
              mStorage = <incomplete sequence \344>,
              mIsSome = 0 '\000'
            }
        netish:
          mChannel:
            mozilla::net::nsHttpChannel 0x7f529ea2a000
              overview:
                mURI:
                  mozilla::net::nsStandardURL 0x7f529eedbf00
                    mSpec:
                      0x7f52bdd6218c "http://example.com/"
                mRequestHead:
                  mozilla::net::nsHttpRequestHead 0x7f529ea2a160
                    mMethod:
                      0x7f52dbf87e08 "GET"
                mResponseHead:
                  mozilla::net::nsHttpResponseHead 0x7f529ea2e870
                    mHeaders:
                      mozilla::net::nsHttpHeaderArray 0x7f529ea2e870
                        mHeaders:
                          nsTArray<mozilla::net::nsHttpHeaderArray::nsEntry> 0x7f529ea2e870
                            mozilla::net::nsHttpHeaderArray::nsEntry 0x7f529e30f408
                              header:
                                0x7f52dbe36c8d "Content-Encoding"
                              value:
                                0x7f529ea2ca8c "gzip"
                            mozilla::net::nsHttpHeaderArray::nsEntry 0x7f529e30f438
                              header:
                                0x7f52dbe6e2ca "Accept-Ranges"
                              value:
                                0x7f529ea2caac "bytes"
                            mozilla::net::nsHttpHeaderArray::nsEntry 0x7f529e30f468
                              header:
                                0x7f52db281292 "Cache-Control"
                              value:
                                0x7f529ea2cacc "max-age=604800"
                            mozilla::net::nsHttpHeaderArray::nsEntry 0x7f529e30f498
                              header:
                                0x7f52dbe36c66 "Content-Type"
                              value:
                                0x7f529ea2caec "text/html"
                            mozilla::net::nsHttpHeaderArray::nsEntry 0x7f529e30f4c8
                              header:
                                0x7f52dc73dfa8 "Date"
                              value:
                                0x7f529ea4248c "Mon, 07 May 2018 20:09:31 GMT"
                            mozilla::net::nsHttpHeaderArray::nsEntry 0x7f529e30f4f8
                              header:
                                0x7f52db280fb4 "Etag"
                              value:
                                0x7f529ea2cb0c "\"1541025663+gzip\""
                            mozilla::net::nsHttpHeaderArray::nsEntry 0x7f529e30f528
                              header:
                                0x7f52dbe6e241 "Expires"
                              value:
                                0x7f529ea424cc "Mon, 14 May 2018 20:09:31 GMT"
                            mozilla::net::nsHttpHeaderArray::nsEntry 0x7f529e30f558
                              header:
                                0x7f52db280fbc "Last-Modified"
                              value:
                                0x7f529ea4254c "Fri, 09 Aug 2013 23:54:35 GMT"
                            mozilla::net::nsHttpHeaderArray::nsEntry 0x7f529e30f588
                              header:
                                0x7f52db1db8b4 "Server"
                              value:
                                0x7f529ea2cb6c "ECS (lga/13A4)"
                            mozilla::net::nsHttpHeaderArray::nsEntry 0x7f529e30f5b8
                              header:
                                0x7f52dc055568 "Vary"
                              value:
                                0x7f529ea2cb8c "Accept-Encoding"
                            mozilla::net::nsHttpHeaderArray::nsEntry 0x7f529e30f5e8
                              header:
                                0x7f52c6f4ada8 "X-Cache"
                              value:
                                0x7f52c737756c "HIT"
                            mozilla::net::nsHttpHeaderArray::nsEntry 0x7f529e30f618
                              header:
                                0x7f52dbe3719e "Content-Length"
                              value:
                                0x7f52c73d973c "606"
                    mContentType:
                      0x7f529ea2cb4c "text/html"
                    mContentCharset:
                      0x7f52db1e8aa4 <gNullChar> ""
                mStatus:
                  nsresult::NS_OK
              context:
                mOriginalURI:
                  mozilla::net::nsStandardURL 0x7f529eedbf00
                    mSpec:
                      0x7f52bdd6218c "http://example.com/"
                mDocumentURI:
                  mozilla::net::nsStandardURL 0x7f529eedbf00
                    mSpec:
                      0x7f52bdd6218c "http://example.com/"
              load:
                mUploadStream:
                  0x0
                mLoadFlags:
                  LOAD_DOCUMENT_NEEDS_COOKIE LOAD_DOCUMENT_URI LOAD_INITIAL_DOCUMENT_URI LOAD_TARGETED LOAD_CALL_CONTENT_SNIFFERS LOAD_CLASSIFY_URI 0x790004
                mLoadGroup:
                  mozilla::net::nsLoadGroup 0x7f52bfd57500
                    mLoadFlags:
                      LOAD_DOCUMENT_NEEDS_COOKIE 0x4
                mLoadInfo:
                  mozilla::net::LoadInfo 0x7f529ee42440
                    mLoadingPrincipal:
                      0x0
                    mOriginAttributes:
                      mPrivateBrowsingId: 0 mUserContextId: 0
    mDocumentURI:
      0x0
  focus:
    mHasFocus:
      false
    mNeedsFocus:
      true

(rr) cbt paste
  0 nsGlobalWindowInner::FreeInnerObjects
    dom/base/nsGlobalWindowInner.cpp:1150
  1 nsGlobalWindowOuter::DetachFromDocShell
    dom/base/nsGlobalWindowOuter.cpp:2177
  2 nsDocShell::Destroy
    docshell/base/nsDocShell.cpp:5474
  3 nsFrameLoader::DestroyDocShell
    dom/base/nsFrameLoader.cpp:1839
  4 nsFrameLoaderDestroyRunnable::Run
    dom/base/nsFrameLoader.cpp:1777
  5 nsIDocument::MaybeInitializeFinalizeFrameLoaders
    dom/base/nsDocument.cpp:6730
  6 detail::RunnableMethodArguments<>::applyImpl<nsIDocument, void (nsIDocument::*)()>(nsIDocument*, void (nsIDocument::*)(), mozilla::Tuple<>&, std::integer_sequence<unsigned long>)
    obj-firefox-debug/dist/include/nsThreadUtils.h:1165
  7 _ZN7mozilla6detail23RunnableMethodArgumentsIJEE5applyI11nsIDocumentMS4_FvvEEEDTcl9applyImplfp_fp0_dtdefpT10mArgumentstlSt16integer_sequenceImJEEEEEPT_T0_
    obj-firefox-debug/dist/include/nsThreadUtils.h:1171
  8 detail::RunnableMethodImpl<nsIDocument*, void (nsIDocument::*)(), true, (mozilla::RunnableKind)0>::Run
    obj-firefox-debug/dist/include/nsThreadUtils.h:1216
  9 nsContentUtils::RemoveScriptBlocker
    dom/base/nsContentUtils.cpp:5635
 10 nsAutoScriptBlocker::~nsAutoScriptBlocker
    dom/base/nsContentUtils.h:3547
 11 nsDocumentViewer::Destroy
    layout/base/nsDocumentViewer.cpp:1769
 12 nsDocShell::Destroy
    docshell/base/nsDocShell.cpp:5464
 13 nsXULWindow::Destroy
    xpfe/appshell/nsXULWindow.cpp:526
 14 nsWebShellWindow::Destroy
    xpfe/appshell/nsWebShellWindow.cpp:776
 15 nsChromeTreeOwner::Destroy
    xpfe/appshell/nsChromeTreeOwner.cpp:294
 16 nsGlobalWindowOuter::ReallyCloseWindow
    dom/base/nsGlobalWindowOuter.cpp:6035
 17 nsCloseEvent::Run
    dom/base/nsGlobalWindowOuter.cpp:5806
 18 nsThread::ProcessNextEvent
    xpcom/threads/nsThread.cpp:1090
 19 NS_ProcessNextEvent
    xpcom/threads/nsThreadUtils.cpp:519
 20 ipc::MessagePump::Run
    ipc/glue/MessagePump.cpp:97
 21 MessageLoop::RunInternal
    ipc/chromium/src/base/message_loop.cc:326
 22 MessageLoop::RunHandler
    ipc/chromium/src/base/message_loop.cc:319
 23 MessageLoop::Run
    ipc/chromium/src/base/message_loop.cc:299
 24 nsBaseAppShell::Run
    widget/nsBaseAppShell.cpp:157
 25 nsAppStartup::Run
    toolkit/components/startup/nsAppStartup.cpp:290
 26 XREMain::XRE_mainRun
    toolkit/xre/nsAppRunner.cpp:4829
 27 XREMain::XRE_main
    toolkit/xre/nsAppRunner.cpp:4974
 28 XRE_main
    toolkit/xre/nsAppRunner.cpp:5066
 29 BootstrapImpl::XRE_main
    toolkit/xre/Bootstrap.cpp:49
 30 do_main
    browser/app/nsBrowserApp.cpp:231
 31 main
    browser/app/nsBrowserApp.cpp:304

(rr) when
Current event: 112750
```
