
## Overview ##
The explicit docs are pretty useful:
https://github.com/bill-mccloskey/searchfox/blob/master/docs/crossref.md

### Contents ###

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
