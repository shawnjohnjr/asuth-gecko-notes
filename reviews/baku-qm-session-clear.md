This seems like a great place to add a larger block comment.  For example:

Clear out QuotaManager storage for principals that have been marked as session only.  The cookie service has special logic that avoids writing such cookies to disk, but QuotaManager always touches disk, so we need to wipe the data on shutdown (or startup if we failed to wipe it at shutdown).  (Note that some session cookies do survive Firefox restarts because the Session Store that remembers your tabs between sessions takes on the responsibility for persisting them through restarts.)

The default value is determined by the "Browser Privacy" preference tab's "Cookies and Site Data" "Accept cookies and site data from websites...Keep until" setting.  The default is "They expire" (ACCEPT_NORMALLY), but it can also be set to "PRODUCT is closed" (ACCEPT_SESSION).  A permission can also be explicitly set on a per-origin basis from the Page Info's "Set Cookies" row.

We can think of there being two groups that might need to be wiped.  First, when the default is set to ACCEPT_SESSION, then all origins that use QuotaManager storage but don't have a specific permission set to ACCEPT_NORMALLY need to be wiped.  Second, the set of origins that have the permission explicitly set to ACCEPT_SESSION need to be wiped.  There are also other ways to think about and accomplish this, but this is what the logic below currently does!




I think we have a potential shutdown phase issue.  sanitizeOnShutdown() is hooked up to Places' ClientsShutdownBlocker which blocks the "profile-change-teardown" phase.  This makes sense for wiping history because:
- Places DB shutdown happens in "profile-before-change" so...
- We don't expect users to be generating new history entries during shutdown.  Although this is probably slightly hand-wavey.
There's some nice documentation at https://searchfox.org/mozilla-central/rev/b28b94dc81d60c6d9164315adbd4a5073526d372/toolkit/components/places/Shutdown.h#17 on what goes on.

Unfortunately, that's not really the right time for QuotaManager clearing to happen because content pages can still get up to stuff unless we close them.  Content processes will be shutdown during "profile-before-change", at least per my notes at https://github.com/asutherland/asuth-gecko-notes/blob/6f827c27f87714a51d6b9a03e17de736484c2838/ipc/SHUTDOWN.md.  I'm not quite clear on what happens

I see 3 high-level ways to address this:
A) Make the clearStoragesForPrincipal
