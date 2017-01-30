## Builds and Conditionals ##

In order to avoid creating code that breaks when it gets to beta/release:
- create a build without --enable-debug in the mozconfig
- in build/defines.sh set EARLY_BETA_OR_EARLIER=0 (previously it was =1)[1]
- in config/milestone.txt changing 53.0a1 to 53.0b1

1: According to comments in
https://hg.mozilla.org/mozilla-central/file/tip/browser/config/mozconfigs/macosx-universal/release
doing export BUILDING_RELEASE=1 can override the failure to zero
EARLY_BETA_OR_EARLIER.

Based on my https://bugzilla.mozilla.org/show_bug.cgi?id=1331949#c3

## JS ##

### SlowScript / WASM issues ###

MOOTED ON TRUNK https://bugzilla.mozilla.org/show_bug.cgi?id=1332312

By default when using "mach", the slow script mechanism will be disabled.  This
disables WASM.  Accordingly, the commandline to use looks something like:

./mach run -P wasm --debugger gdb --slowscript

Noting that it seems like I'm experiencing some serious "gdb session locks up"
problems in the variant where I've been trying this...

### Dumping Stacks ###

calling DumpJSStack() works a lot of the time

If that won't work, but you can get a cx somehow:
* call fputs(stderr, xpc_PrintJSStack(cx, true, true, false))
* call js::DumpBacktrace(cx)

The reason the above might happen per bz:
"...when JS called 'processNextEvent'"
"we pretend like no JS is running before running the next runnable"

## IPC ##

### IPC Logging ###

On a debug build, use `MOZ_IPC_MESSAGE_LOG=1`

Advisable to use with tee and stderr redirection, so

MOZ_IPC_MESSAGE_LOG=1 ./mach stuff |& tee /tmp/ipc.log
