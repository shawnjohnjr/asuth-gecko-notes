## Thread Safety ##

### NS_ASSERT_OWNINGTHREAD ##

From nsISupportsImpl.h:
* NS_DECL_OWNINGTHREAD
* NS_ASSERT_OWNINGTHREAD

## Structured Cloning ##

Wire Representations:
* Primary serialized rep (without transferables):
  * ipc::StructuredCloneData: IPC-aware StructureCloneHolder subclass that uses
    JSStructuredCloneData for storage and underlying serialization.
  * SerializedStructuredCloneBuffer: IPC-aware type that wraps
    JSStructuredCloneData providing operator=
  * JSStructuredCloneData (IPCMessageUtils.h)
* Rep w/transferables:
  *
