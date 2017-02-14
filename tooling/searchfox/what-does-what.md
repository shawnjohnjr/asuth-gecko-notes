

## Random Scripts ##

### root ###
* findall.py - half-broken tree-walker that currently just prints probably
  non-binary files that it finds, tallying (but not currently printing) sizes
  by extension, and avoiding recursing into build and .git directories (but not
  .hg)
* find-js.py - similar to findall, prints .js and .jsm files as it finds them,
  but tallies all observed file sizes (and does nothing with it), avoids
  recursing into objdirs or test or .git directories.
* idl-analyze.py -
* js-analyze.js -
* lib.js -
* output-dir.js -
* output-help.js -
* output.js -
* output-template.js -
* setversion.js - Used by js-analyze.sh as snippet to be evaluated by "js" shell
  execution to force use of JS v1.7 so old-style generators are allowed.
  Precedes actual scripts.

### scripts/mkindex.sh family ###

mkindex.sh uses a whole bunch of others.  It's the prime mover, directly or
indirectly using everything but copy-objdir-files.py, generate-config.sh,
indexer-setup.py, nginx-setup.py.

Given <config-repo-path> <config-file> <tree-name>:
* $CONFIG_REPO/$TREE_NAME/find-repo-files - recurses through tree, building up
  newline-delimited "repo-files", "repo-dirs", "js-files", "idl-files" in the
  index dir.
* mkdirs.sh - Consumes repo-dirs file and produces parallel directory
  hierarchies under $INDEX_ROOT/{file,dir,analysis}.
* $CONFIG_REPO/$TREE_NAME/build - Creates a mozconfig at INDEX_ROOT/mozconfig
  with debug/optimize, and using indexer-setup.py to generate CC/CXX variables
  to build using the clang-plugin, configured to dump the analysis in
  $INDEX_ROOT/analysis.
* find-objdir-files.py - Analogous to find-repo-files, recurses
  $INDEX_ROOT/analysis/__GENERATED__ to create $INDEX_ROOT/objdir-{files,dirs}
* objdir-mkdirs.sh - Analogous to mkdirs.sh, creates the parallel __GENERATED__
  structure under $INDEX_ROOT/{file,dir}.  (analysis was the source and so
  doesn't need anything to be generated.)
* js-analyze.sh -

* build-codesearch.py -
* copy-objdir-files.py -
* crossref.sh -
* generate-config.sh -
* idl-analyze.sh -
* indexer-setup.py -
* lib.py -
* load-vars.sh -
* nginx-setup.py -
* output.sh -
* read-json.py -
