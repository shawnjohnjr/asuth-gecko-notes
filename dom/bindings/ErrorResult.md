Common/Unified error abstraction.

## Overview ##

Canonical storage is mResult with special error types indicating whether the
union types are in use.  (There is a debug-only mUnionState that provides
redundancy to enable assertions, including a HasNothing state for when there's
just a normal nsresult in use.):
* HasMessage:  TErrorResult<CleanupPolicy>::Message* mMessage.  Stores:
  * nsTArray<nsString> mArgs
  * dom::ErrNum mErrorNumber
* HasDOMExceptionInfo: DOMExceptionInfo* mDOMExceptionInfo.
* HasJSException: JS::Value mJSException.

What common error idioms use:
* ThrowTypeError: Message with errorType=mResult=NS_ERROR_TYPE_ERR, and the
  passed-in errorNumber and string args passed in the message.

### How the errors hit content ###

Note that the type is magic and will assert if it doesn't think you properly
processed the error.

nsresult consumers/exposers usually consume via StealNSResult() or explicit
SuppressException() if they checked Failure() and don't really care.

When interacting with content (from DOM-space) ToJSValue(JSContext*,
ErrorResult&, JS::MutableHandle<JS::Value>) can create an exception.  Its
mechanism is simple: It uses ErrorResult::MaybeSetPendingException(cx) then
steals the exception via JS_GetPendingException(cx, value), then clears it via
JS_ClearPendingException.
* Message case ends up in JS_ReportErrorNumberUCArray

Message
