
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

### with vagrant

Just mozilla-central, no try
```
KEEP_WORKING=1 /vagrant/infrastructure/indexer-setup.sh ~/mozilla-config ~/mozilla-index mozilla-central
```

With try:
```
KEEP_WORKING=1 /vagrant/infrastructure/indexer-setup.sh ~/mozilla-config ~/mozilla-index mozilla-central https://queue.taskcluster.net/v1/task/Q6NALCiWRxSgTuhZ2DkvLA/runs/0/artifacts/public/build/target.json
```

Now run the index!
```
/vagrant/infrastructure/indexer-run.sh ~/mozilla-config ~/mozilla-index mozilla-central
```

WHOOPS, VAGRANT ISSUES;
~/vmirror/infrastructure/indexer-run.sh ~/mozilla-config ~/mozilla-index mozilla-central



Now make sure the server is all configured.
```
mkdir -p ~/web-logs
/vagrant/infrastructure/web-server-setup.sh ~/mozilla-config ~/mozilla-index ~/web-logs
```

Now run the server.
```
/vagrant/infrastructure/web-server-run.sh ~/mozilla-config ~/mozilla-index ~/web-logs
```
