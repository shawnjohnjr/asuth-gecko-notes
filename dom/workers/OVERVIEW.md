## Registrations ##

ServiceWorkerRegistrar keeps registrations in newline-delimited
"serviceworker.txt" in profile directory.  File "header" is version string
(currently 4), followed by repeating blocks of [scope, script, guid].

In the parent process, the SWM directly asks the SWR for the list of
registrations.  (Which the SWR loaded at profile-after-change time.)
