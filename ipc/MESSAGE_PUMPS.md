Note: I think these notes pre-date the unification khuey did.

## MessagePump ##

base::MessagePumpDefault (with well-documented logic in base::MessagePump's
header) has a regular cycle of: DoWork(), DoDelayedWork(), (continue if we did
some work), DoIdlyWork().  Gecko's MessagePump alters this flow so that the
cycle is: NS_ProcessNextEvent(), DoDelayedWork(), (continue if did work),
DoIdleWork(), NS_ProcessNextEvent().

It changes things so that DoWork() is performed by the singleton DoWorkRunnable,
which it schedules.  Additionally, DoWorkRunnable is an nsITimerCallback which
when fired invokes DoDelayedWork() on the pump (which internally invokes the
delegate and then potentially re-schedules things.)  The Main run loop also
includes some additional timer-related logic.

### MessagePumpForChildProcess ###

Invokes XRE_RunAppShell.  Unclear if the appshell for child processes has been
stubbed out to not spin; it certainly looks like it's expecting to fall through
into MessagePump::Run of the base class.  Per the comment in XRE_RunAppShell it
seems like the point is mainly to spin up the appshell so XPCOM gets spun up.

### MessagePumpForNonMainThreads ###

Seems mainly concerned with creating its own delayed work timer and draining the
DoWork backlog before entering what is essentially the normal run loop
copied-and-pasted.

### MessagePumpForNonMainUIThreads ###

Windows-specific runloop that adds windows event loop gunk.  I don't know enough
about these, but intuition is probably sufficient for now.
