## Terminology ##
(not sure idioms is right here)

* Windows:
  * HTML Spec "Browsing Context" == Gecko "outer window" and docshell
  * HTML Spec "Window" == Gecko "inner window"

## Thread Safety ##

### NS_ASSERT_OWNINGTHREAD ##

From nsISupportsImpl.h:
* NS_DECL_OWNINGTHREAD
* NS_ASSERT_OWNINGTHREAD

NS_DECL_OWNINGTHREAD also happens "for free"/"as a side effect of"
NS_INLINE_DECL_REFCOUNTING_WITH_DESTROY and its more common form,
NS_INLINE_DECL_REFCOUNTING.  This is because there's also the aptly named
NS_INLINE_DECL_THREADSAFE_REFCOUNTING, and in DEBUG builds the non-threadsafe
refcounting needs to be able to assert if you try and manipulate refcount on
more than one thread.

## Structured Cloning ##

Wire Representations:
* Primary serialized rep (without transferables):
  * ipc::StructuredCloneData: IPC-aware StructureCloneHolder subclass that uses
    JSStructuredCloneData for storage and underlying serialization.
  * SerializedStructuredCloneBuffer: IPC-aware type that wraps
    JSStructuredCloneData providing operator=
  * JSStructuredCloneData (IPCMessageUtils.h)
* Rep w/transferables:
  * ClonedMessageData (DOMTypes.ipdlh)
* Rep w/persistables:
  * SerializedStructuredCloneReadInfo is SerializedStructuredCloneBuffer w/blobs
    with nullable PBlob/PBackgroundMutableFile.

scratch:
  struct ClonedMessageData
  {
    SerializedStructuredCloneBuffer data;
    PBlob[] blobs;
    MessagePortIdentifier[] identfiers;
  };

  struct MessagePortMessage
  {
    SerializedStructuredCloneBuffer data;
    PBlob[] blobs;
    MessagePortIdentifier[] transferredPorts;
  };
