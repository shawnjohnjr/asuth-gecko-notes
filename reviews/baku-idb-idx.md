DeserializeIndexValueHelper:
- Cross-thread blocking helper with monitor notification.
- Exists to address the fundamental issue that we need a global that can handle
  cycle-collected DOM bindings.  Attempting to deserialize with the
  CreateIndexOp::ThreadLocalJSContext was not good enough.  Using a Worker
  would be cleaner, but is future work.
- Empty data still results in fast-path that avoids the main-thread transit.

DeserializeUpgradeValueHelper:
- Replaces UpgradeDeserializationHelper, with a similar rationale to
  DeserializeIndexValueHelper and accordingly a similar implementation that also
  uses SandboxHolder, etc.
- The big difference is PopulateFileIds is refactored out of the
  on-IO-thread UpgradeFileIdsFunction::OnFunctionCall into the on-main-thread
  helper.  (The structured clone deserialization and interpretation of the
  IDB StructuredCloneReadInfo is all moved there.)

Changes to ValueDeserializationHelper:
- CreateAndWrapBlobOrFile gains logic to create the missing Blob instance in
  part 1.  We don't really need to be creating the File here other than to avoid
  surprise later.  The important meta-info for the blob/file for index creation
  for the evaluation of any key paths is set lower down in the method by the
  calls to SetLazyData.  This logic is merely approximating the net result
  of ActorParent.cpp's SerializeStructuredCloneFiles and ActorChild.cpp's
  DeserializeStructuredCloneFiles which:
  - Create a FileBlobImpl wrapping the result of GetCheckedFileForId() and
    annotated with SetFileId().  The file id is used exclusively for unit tests
    that need to pierce the abstraction barrier via superpowers.  Since the blob
    is never surfaced that we produce, the lack of setting the fileid doesn't
    matter, although maybe we should... noting in review.  Then IPC serialize.
  - In the child, performs IPC deserialization and then does Blob::Create which
    amounts to File::Create, so it ends up the same.
- CreateAndWrapMutableFile gains logic to change the StructuredCloneFile mType
  to be eMutableFile if it was eBlob in part 2.  This is part of the
  DeserializeUpgradeValueHelper migration which depended on the structured clone
  read to tell us what the type of the blob is.
- CreateAndWrapWasmModule gets dummy-object logic in part 4.  This relaxes the
  assertion that mWasmModule is present that only holds true if
  ActorsChild.cpp's DeserializeStructuredCloneFiles populated it.  We don't
  really need the assertion for correctness, it's just more of an anti-crash
  guard, so the failsafe and existing functionality tests should cover that
  guarantee.

Removed CreateIndexOp::ThreadLocalJSContext : NormalJSContext (part 1):
- Superseded by SandboxHolder.
Removed NormalJSContext (part 2):
- This created a custom context with a basically no-op global.

SandboxHolder:
- Correctly uses StaticRefptr.

GetFileForFileInfo is moved from a soupy existence in ActorsParent.cpp to live
next to similar methods in FileInfo.{h,cpp} so that IDBObjectStore.cpp's
ValueDeserializationHelper::CreateAndWrapBlobOrFile can reify the blob given
the mFileInfo.
