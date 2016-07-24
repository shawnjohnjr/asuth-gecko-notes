## Understanding IPC/Channel Flow ##

Trace the flows of fetch(x) leading to http channel X, intercept channel X,
affiliated IPC messages, through to completion.

Since conditional breakpoints already imply breaking, might as well just extract
traces at all call sites and stitch together afterwards.  This also potentially
avoids needing to be exceedingly clever with rr and multiple processes.  Just
run once for the parent process and once for each child process.

Potentially run in a debug build so we have IPC logging and can breakpoint on
LogMessageForProtocol.  On the send side we can then walk up a frame to gain
access to the message payload and pick whatever we want out of it.
