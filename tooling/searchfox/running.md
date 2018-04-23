
## Testing Searching

```
python router/router.py index/config.json
```

## Testing the indexers

### clang

This is basically what the build does:
```
# mozconfig is at $INDEX_ROOT/mozconfig
export MOZCONFIG=~/rev_control/fgit/searchfox/index/mozilla-central/mozconfig
# mach gets run from $FILES_ROOT
cd ~/rev_control/fgit/searchfox/index/mozilla-central/gecko-dev
./mach build
```
With that MOZCONFIG configured, mach build should be usable.

### IDL

scripts/idl-analyze.sh index/config.json mozilla-central *filterspec*

## Regenerating Certain Stages

If you have indexer changes, I then have my dumb forked scripts that are a
cut-down version of mkindex.sh:

```
scripts/crossref-index.sh config index/config.json mozilla-central

scripts/output-index.sh config index/config.json mozilla-central
```
