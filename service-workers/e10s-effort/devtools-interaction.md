## What ##

about:debugging, living at devtools/client/aboutdebugging

Actor API consumption:
* tabs: components/tabs/panel.js uses client.mainRoot.listTabs()
* workers:
  * modules/worker.js has getWorkerForms which retrieves:
    * registrations: client.mainRoot.listServiceWorkerRegistrations()
    * workers from the parent process: client.mainRoot.listWorkers()
    * workers from all immediate child processes by iterating over
      client.mainRoot.listProcesses() and invoking listWorkers for each of
      them (indirectly via client.request)
  * distinguishes between nsIWorkerDebugger.TYPE_SERVICE,
    nsIWorkerDebugger.TYPE_SHARED, with else fallback to "other".

With the server bits being:
* devtools/server/actors/webbrowser.js (parent process):
  * WorkerActorList from ./worker.js
    * confusion: slurped out of nsIWorkerDebuggerManager, but this does not
      expose the entirety of the set of workers known to the RuntimeService,
      just those with registered debuggers?  I think...
  * ServiceWorkerRegistrationActorList from ./worker.js
    * slurped out of nsIServiceWorkerManager
