https://bugzilla.mozilla.org/show_bug.cgi?id=1339081

Most comments/analysis made directly on the bug.

## Part 14

OriginParser::Parse has been updated to have an out-string, aOriginalSchema.
If the parsed schema is "app", the spec is intentionally altered to use a spec
of "http" instead, but with aOriginalSchema taking on the "app" schema.  This
choice was made because returning false means


QM::EnsureOriginIsInitialized had a bail-out previously if the origin parser
refused to parse

MaybeUpdateSchema hacks the http:// back to being app://.
