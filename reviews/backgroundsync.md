## Database/Storage ##
Using backgroundsync file names since they're somewhat more split/renamed to be
more accurate.

Same (modulo smaller things that can be parameterized):
* StorageAction.h/cpp (from Action.h/cpp)
* DBAction: Opportunistic connection reuse, logic related to opening and reusing
  databases, possibly re-creating from schema.
* DBConnection (Connection in dom/cache): exists to wrap mozIStorageConnection
  in order to provide an automatic incremental vacuum on connection close.
  Otherwise manual boilerplate-y wrapper.
* DBSchemaUtils: InitializeConnection with WAL/page/growth parameters and sanity
  checks.  Genericized CreateOrMigrateSchema that takes SQL,
  currently-limited-to-one table-names.  IncrementalVacuum with sanity checks.
  Validate with DEBUG-only exact parameterization of expected tables and indices
  and SQL in cache case, which could probably be pulled out to be non-debug
  exact characterization of schema.  Somewhat altered in backgroundsync to take
  single parameter.
* QuotaClient: Recursive file-walker with some limited hard-coded filename
  prefixes.  dom/cache has extra directory check logic to handle existince of
  "morgue"

Domain specific
* DBSchema: Where all the actual specific database logic happens.

INVESTIGATE MORE, but cargo culted

* StorageContext.h/cpp (from Context.h/cpp)
