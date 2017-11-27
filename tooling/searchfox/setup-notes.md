
Largely, just following the local development instructions, I did the following
in my checkout of searchfox:

```
infrastructure/indexer-provision.sh
```

The install docs then suggest using update.sh, but the script is very confusing
because it wants to check-out "mozsearch", which is now "searchfox", so it would
be a recursive checkout.  Instead we do the relevant seeming bits, noting that
I enforced
```
pushd clang-plugin
make CXX=clang++
popd

pushd tools
git submodule init
git submodule update --recursive
cargo build --release --verbose
```





```
git clone https://github.com/bill-mccloskey/mozsearch-mozilla.git config

mkdir index

infrastructure/indexer-setup.sh config index
infrastructure/indexer-run.sh config index
```

My second run of indexer-setup.sh screwed up...

re-indexing stuff:
```
scripts/mkindex.sh config scratch/config.json mozilla-central
scripts/mkindex.sh config scratch/config.json comm-central
```
