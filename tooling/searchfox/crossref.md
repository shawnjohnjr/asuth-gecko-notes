
## Overview ##
The explicit docs are pretty useful:
https://github.com/bill-mccloskey/searchfox/blob/master/docs/crossref.md

## Generation ##

The crossref logic slurps analysis data via read_analysis with read_target.
(Contrast with output-file.rs which uses read_source.)

## Byproducts ##

The crossref binary produces the following ${index_path} files:
* crossref: The per-symbol information
* jumps:
* identifiers: Mapping from human-readable identifiers with and without their
  namespacing bits to their symbol names.  Binary-searched by router.

### crossref ###

As per the docs, the structure is a dictionary whose keys are one of:
* Declarations
* Definitions
* Uses
* Assignments
* IDL

The values are an array of entries of the form `{ path, lines[] }` where each
line is an object with the following fields:
* bounds: [zero-based symbol start column, zero-based end column]
* context: Human-readable qualified (type) name of the enclosing context.  For
  example the function/method in which the call occurs, or the class a method
  is being defined on, etc.
* contextsym: The (mangled) symbol name version of the context, only available
  for C/C++.  This strictly has more information which is desired for dealing
  with overloaded functions.
* line: the contents of the line
* lno: line number

### jumps ###
Each line in the file is an array of:
* id: The searchfox symbol, which includes the weird "#-#" things.
* path: Repo-relative file path
* lineno: Yeah, the line number.
* pretty: The pretty symbol.

analysis.rs' read_jumps is used to build a HashMap from the id/symbol to the
Jump struct which holds all of that info.

These jumps are used when format.rs' format_code is building up ANALYSIS_DATA.
This is the same place where the "search" variants are built by processing the
"source" labeled items in the analysis file.  (Loaded via read_analysis with
read_source.)
