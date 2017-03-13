# API Notes

## Service Workers ##

### register() ###

Register's resolution timeline looks like
- Rejected early in "Start Register" for things like a bogus scriptURL (parse
  failure, bad scheme, illegal characters), bogus scopeURL (same).
- Resolved in a fast-path in the Register algorithm (inside a job) if
  Get Registration returns non-null and the script URL of the registration's
  newestWorker is the same as that of the request.  (This is independent of
  update checks.)
- (Theoretical fast-path in step 8 if byte identical cannot happen in the
  register case because we already fast-pathed out above on the scriptURL which
  is a precondition for byte identical.)
- Slowest resolve is where register handed off to update handed off to install,
  which resolves at step 7.  At this point the registration will be
  "installing".

So case-wise:
- Rejection on errors.
- Resolved with a registration where the given script URL may be associated with
  an active, waiting, or installing worker if the script URL was already
  registered.  The fast-path check is based on the newestWorker algorithm.  If
  (non-register) updates are happening, the script URL may be involved with
  multiple registrations even.
- Resolved with a registration where the given script URL is the installing
  service worker if the script URL was not already registered.
