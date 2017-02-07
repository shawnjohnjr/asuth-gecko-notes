https://bugzilla.mozilla.org/show_bug.cgi?id=1324836

Review of changes from origin OriginScope implementation I'd reviewed so that
the home-brew discriminated union now uses the MFBT Variant template.

Previously there was only a single custom type, OriginAndAttributes, with
OriginAttributesPattern used directly for ePattern and nsCString used directly
for ePrefix, plus a dummy void* for eNull.

Now there's 4 explicit struct types.  Each of them supports copy construction.

Special things:
* OriginString defined as subclass of nsCString that seems to primarily exist to
  provide a convenient BeginsWith helper.


### Subclass Minutae

OriginString: gets default copy/move, likely ends up as assignment.
* Has const nsACString& constructor, suppresses default constructor.
* Has operator=(const nsACstring&) which does not count as a copy operation, so
  it does not suppress default copy/move/destructor.
* Note that nsCString and friends do not define move constructors/operators, so
  it likely ends up as assignment.

Origin: move suppressed
* has custom copy constructor, should suppress move.

Pattern:
* has custom copy constructor, should suppress move

Prefix:
* does not have copy constructor; the nsACString& is a different type.
