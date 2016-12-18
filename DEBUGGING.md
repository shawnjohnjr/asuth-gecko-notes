## JS ##

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

MOZ_IPC_MESSAGE_LOG ./mach stuff |& tee /tmp/ipc.log