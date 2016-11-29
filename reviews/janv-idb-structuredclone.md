## Part 1: Use a type in StructuredCloneFile instead of a boolean ##

Duplicately defined enum oddness gets cleaned up by part 1 of wasm.
* `struct StructuredCloneFile` in IndexedDatabase.h defines:
  enum Type { eBlob, eMutableFile };
* `struct ObjectStoreAddOrPutRequestOp::StoredFileInfo` contains a duplicate
  enum def.

The string serialization of the ids is negative for eMutableFile, positive for
eBlob.  This is metadata that is always carried with the id; the "file" table
only exists to store refcounts (using triggers) and does not have metadata.

2 AppendInt call-sites:
* UpgradeFileIdsFunction::OnFunctionCall. Specialized upgrade function that
  extracts the referenced file id's from the structured clone that's being upgraded.

* ObjectStoreAddOrPutRequestOp::DoDatabase: converted to Serialize call in part 2.

1 sign-based type inference sites:
* DatabaseOperationBase::GetStructuredClon: converted to Deserialize call in part 2.

## Part 2: Refactor file ids handling for better expandability ##

Negative starts getting handled as just a magic prefix character in
DeserializeStructuredCloneFile.


## Part 3: Implement an input stream wrapper around structured clone data ##

restating: Composes on top of JSStructuredCloneData which uses mfbt's
BufferList.

The ReadSegments loop count bailing is correct because Advance() automatically
traverses to the next buffer so RemainingInSegment() will always be >0.

## Part 4: Update keys directly in the structured clone buffer ##

janv says:
```
This is needed because when we compress externally stored structured clone data
we don't need to create one big flat buffer. We read from structured clone
buffers directly, so the key must be updated there. The writing is a bit
complicated since we can cross buffers after each char of the key in theory.
```

restating:
* writes the autoincrementing key into an in-memory buffer
  (writeUint64 is from mfbt/EndianUtils.h which is strictly about writing into
  void* memory.)
* in this patch the data still unconditionally gets exploded to flatCloneData,
  but this is addressed in part 6.  The flatCloneData is only created if we're
  not in overflow and are directly/immediately binding.

## Part 5: Add a data threshold pref ##

janv says:
```
I decided to have 1 MB threshold for now. We can lower it once we are confident
there are no bugs related to this new feature.
```

**respond**: Makes sense to set it high initially; I'm assuming we'd otherwise want
it to be some multiple of the page size .

mooted: I had a comment about using the cached preference variants, but it turns
out that

## Part 6: Core changes for storing structured clone data outside of the database ##

janv says:
```
::: dom/indexedDB/ActorsParent.cpp
@@ +25726,5 @@
> +  if (mDataOverThreshold) {
> +    // Higher 32 bits reserved for flags like compressed/uncompressed. For now,
> +    // we don't compress externally stored structured clone data since snappy
> +    // doesn't support streaming yet and we don't want to allocate another
> +    // large memory buffer for that.

This comment is updated in following patch for compression.
```

restating:
* BlobOrMutableFile gains a null_t union type.  (Which is in some ways odd
  because NulltableMutableFile can already be null_t.)  Fixed in webasm part 2.
* schema gets bumped to 25 from 24 and enforced.
* both enums gain eStructuredClone
* eStructuredClone gets prefix encoded with "." (whereas mutable was previously
  "-" and normalized to that being a prefix rather than numeric parsing.)
* The structured clone storage column that was previously a blob (invariant)
  is now either an integer (external blob) or blob (same old, same old).
  * The integer encodes the index, gains a flag next patch.
  * The read occurs synchronously into the structuredcloneinfo (Via WriteBytes).
  * The data-type already had BLOB affinity which means no coercion, so the
    existing column definition is more than fine.  **maybe comment on this?**
* mDataOverThreshold is based on snappy's worst-case resulting output size.

useful context:
* the fallible array usages are guarded via SetCapacity calls higher up.

pending comments:
* On UniquePtr and free().


## Part 7: Compress externally stored structured clone data ##

restating:
* the database integer encoding for where the structured clone used to go is:
  * upper 32 bits: flags, with them being: H[compressed]L.
  * lower 32 bits: index of the file in the (serialized) file list.

## Part 8: Disable externally stored structured clone data in file tests ##

janv says:
```
We need this just in case someone runs all the tests with the threshold set to 0.
In that case all structured clones need to allocate a file id.
However, file tests have static assumptions about blob file ids, so file tests don't pass.
It would be really hard and messy to fix that.
```

## Part 9: Add a new test for externally stored structured clone data ##


# WASM #

## Part 3: Core changes for WebAssembly module serialization including a test ##

restating:
* adds new file list prefix tags:
  * '/' for wasm bytecode
  * '\\' for wasm compiled

## Part 6 ##

restating:
* DeserializeStructuredCloneFile gets cleaned up with switch-based prefix
  mapping directly to FileType type.
* ScopedPRFileDesc magical self-closing

pending comment:
* can mValid have a comment?

## Part 8: Enable storing WebAssembly modules in IndexedDB ##

notes:
* WasmModulePreprocessInfo: struct of SerializedStructuredCloneFile[]
* Various GetStructuredCloneReadInfo* variations modified to pass bool* aHasWasm
* SerializeStructuredCloneFiles gaining aForPreprocess:
  * Causes non-wasm file types to be skipped.
  * Reciprocally, when in those types and !aForPreprocess, rather than
    processing as a BlobImplStoredFile, a null_t is exposed, similarly to if the
    file was not valid.


restating:
* TransactionDatabaseOperationBase refactorings:
  * Introduce explicit InternalState mInternalState state-machine progression.
    * Conceptually new states SendingPreprocess and WaitingForContinue and
      additional invariant-check/machinery states Initial and Completed.
      (Previously there were just de facto DatabaseWorker in the form of
      RunOnConnectionThread and SendingResults in the form of RunOnOwningThread)
    * Methods for override in subclasses:
      * HasPreprocessInfo (defaults to false)
      * SendPreprocessInfo
  * Add StartOnConnectionPool helper, replacing direct gConnectionPool->Start
    calls, but more importantly automatically advancing mInternalState to
    InternalState::DatabaseWork.  Note that this was an idiomatic code path to
    begin with; most ops were dispatched via DispatchToConnectionPool().
    * And DispatchToConnectionPool now defers dispatch to the state machine via
      Run() which in turn calls SendToConnectionPool().
  * Related to both of the above, TransactionDatabaseOperationBase::Run is now
    a dispatch based on mInternalState and the RunOn* methods have been made
    specific.

feedback:
* PBackgroundIDBRequest PreprocessResponse having an `nresult` branch apparently
  for errors is weird.  **crosschecking getall changes now**

## Part 9 ##

## Part 10 ##

# Bug 1312808 : Persisted Version Upgrades of WASM #

## Low-level Change Summary ##

ActorsChild.cpp:
* DeserializeStructuredCloneFiles simplification:
  * The two separate switch branches for StructuredCloneFile::eWasmBytecode and
    StructuredCloneFile::eWasmCompiled are being combined.
  * *Assertion Change!*  Previously the type for eWasmBytecode was asserted to
    be TPBlobChild; eWasmCompiled could be that or Tnull_t.  Now both are
    required to be eWasmBytecode.
  * Accordingly, the switch that enabled different behavior for null and
    non-null went away.
  * Other changes that assume the compiled file blob is always available.

ActorsParent.cpp:
* FileHelper abstraction introduced:
  * CheckWasmModule:
    * Existing check logic migrated in here; takes the form of opening the file
      descriptor and getting the buildid, then handing both off to
      JS::CompiledWasmModuleAssumptionsMatch.  If they match, early return.
    * Mismatch continues onward with new logic!
    * Also opens the bytecode file's descriptor.
    * Deserializes the wasm module.
    * Re-compiles the module: get size, allocate contiguous buffer, serializes
      into it.
    * Writes it:
      * Creates ByteInputStream inputStream
      * Creates [newFile, newJournalFile] pair
      * Uses FileHelper::CreateFileFromStream to write it to disk
      * Uses FileHelper::ReplaceFile to replace compiledFile with newFile.
      * Uses FileHelper::RemoveFile to clean up newFile, newJournalFile.
  * ObjectStoreAddOrPutRequestOp::DoDatabase:
    * file directory and journal directory ensure

## High-level ##

Previous handling of out-of-date compiled blobs:
- Would check at open time and send down the null_t compiled blob.
