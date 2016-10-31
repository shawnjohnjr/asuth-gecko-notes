## IPC ##

### IPC Logging ###

On a debug build, use `MOZ_IPC_MESSAGE_LOG=1`

Advisable to use with tee and stderr redirection, so

MOZ_IPC_MESSAGE_LOG ./mach stuff |& tee /tmp/ipc.log
