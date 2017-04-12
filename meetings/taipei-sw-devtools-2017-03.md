## Enabling / Disabling Bypasses, etc. ##

Seems to primarily happen by manipulating the docshell and its defaultLoadFlags.

`_setCacheDisabled`:
http://searchfox.org/mozilla-central/source/devtools/server/actors/tab.js#1048

example consumer, `toggleDevtoolsSettings`:
http://searchfox.org/mozilla-central/source/devtools/server/actors/tab.js#1003

nsIDOMWindowUtils has serviceWorkersTestingEnabled, the setter is:
http://searchfox.org/mozilla-central/source/dom/base/nsDOMWindowUtils.cpp#3974
It propagates to a setter on the outer window.

Its getter impacts:
- SWM::Register treats all origins as "authenticated"/secure.
  (IsFromAuthenticatedOrigin).
- nsGlobalWindow::GetCaches similarly does forceTrustedOrigin.
- SWM::GetAllClients: same deal, treat as secure.
- WorkerPrivate::GetLoadInfo puts it on the loadinfo as
  mServiceWorkersTestingInWindow, which then is exposed as
  WorkerPrivate::ServiceWorkersTestingInWindow
  http://searchfox.org/mozilla-central/source/dom/workers/WorkerPrivate.h#862
