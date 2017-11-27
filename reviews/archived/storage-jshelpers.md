https://bugzilla.mozilla.org/show_bug.cgi?id=1390489

Part of the https://bugzilla.mozilla.org/show_bug.cgi?id=1389510 family.
Others include:
* landed: https://bugzilla.mozilla.org/show_bug.cgi?id=1390147

### General Problem Space Notes

We provide magic "row" and "params" JS helpers on mozIStorageStatement.  These
are exposed via mozilla::storage::StatementClassInfo::GetScriptableHelper which
defines and returns its `static StatementJSHelper sJSHelper;`

The helper provides a Resolve hook that binds the (static) `stepFunc` function
to "step" on demand, and invokes getRow() and getParams(), defining "row" and
"params" properties, respectively, from their returned values, on demand.

For their part getRow()/getParams() pull the mStatement{Row,Params}Holder off
the Statement if it already exists.  If one didn't exist, they:
- Create the new Statement{Row,Params} instance.  These are the actual JS magic
  pieces that previously used GetProperty and now use the WebIDL NamedGetter
  mechanism.
- Create the Statement{Row,Params}Holder instance with a reference to the row.
  This is saved off on the Statement.


### Minutae/old

StatementRowHolder and StatementParamsHolder subclass StatementJSObjectHolder
which:
- subclasses nsIXPConnectJSObjectHolder.
- has member field nsCOMPtr<nsIXPConnectJSObjectHolder> mHolder;

Each of their destructors:
- Asserts they are on the main thread.
- QI's their nsIXPConnectWrappedNative and then does do_QueryWrappedNative to
  get back to their root mozIStorageStatementParams/mozIStorageStatementRow
  interface.  They then static_cast the raw pointer (via get()) to
  StatementParams/StatementRow,  which lets them...
- null-out the type's `Statement *mStatement`.

#### Theory? / Stuff going on:

Ownership improvement:
- StatementRowHolder gains a strong `RefPtr<StatementRow> mRow`.
- StatementRow::* loses NS_ENSURE_TRUE(mStatement, NS_ERROR_NOT_INITIALIZED).

This makes sense if:
- In the olden days, the `row` "tear-off" (right term?) could become invalidated
  if references to the underlying statement were discarded.
- With this change, the row holder creates a strong circular reference cycle
  between the statement and the holder so that the `row` tear-off needs to be
  collected... **but** it's not clear that the row's destructor does anything?
