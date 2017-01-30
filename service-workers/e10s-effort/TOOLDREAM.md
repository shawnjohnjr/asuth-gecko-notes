## treeherder try push results scraper helper ##

Alternate treeherder view reminiscent of ArbPL test failure tree structure that
understands touched (or explicitly interesting) tests and can show both pending
and results.  Plus perhaps diff-view of their differing log results.

## Spec algorithm implementation control flow understanding based on comments ##

A lot of Gecko code includes references to algorithm callouts in the code.
If these comments are sufficiently structured with the algorithms sufficiently
called out, it would be possible to provide a combined view that lists the
current algorithm with the spec and where in the code we claim to be
implementing those steps.

## Assisted code reading ##

Starting at the top of a file and reading through is frequently not the best way
to figure out what's going on.  Tooling could help provide guidance from within
traditional code-reading scenarios such as dxr/searchfox raw code listings or
in their editor.

(NB: This has some conceptual overlap with the doccelerator effort whose UI was
based on tiddlywiki where per-method/whatever docs were basically cards that
were inserted into a working set of cards.  Individual cards could then be
annotated with comments or things in that idiom.  While not a bad approach for
what would otherwise be a javadoc-style encyclopedic UI, that's obviously not
a natural interface to someone who is )

### Through superclass annotations ###

For example, ServiceWorkerJob subclasses do their main logic in AsyncExecute.
If ServiceWorkerJob is annotated, then in a file implementing the class (and
for which we might be able to determine it is the class that matters based on
it being what's exposed in the corresponding header file, or just that it can
be seen to dominate the other classes in the file through control flow usage.

### By creating a narrative traversal of the file based on apparent flow ###

Based on the above, if we can know what the meaningful entry points are, and we
can figure out which methods/helper classes are exclusively covered by entry
points, we can isolate those code-paths into their own "chapters" that are
linearized through the control-flow path.

In cases with explicit transfer of control flow using runnables, the links can
be followed in a somewhat straightforward fashion.  In cases with a state
machine, some level of interpretation based on when the state variable is
changed and read would be needed to re-establish the graph.

## DXR local call graph info w/parameter classification ##

For example, in the case of SpawnWorkerIfNeeded, one of the arguments was
`nsIRunnable* aLoadFailedRunnable`.  Some callers explicitly provide a runnable
(runnable newed in the method), some explicitly do not (nullptr), and some pass
through a runnable from their caller.

Being able to click on the runnable argument and hit a button to get a dot-style
graph (or just a grid?) that shows the three categories and call-sites, plus
ideally a depth=1 recursive callers situation for the caller-propagates case
would be killer.

## Overviews of State w/Multiprocess Linkage ##

Use GDB to dump JSON representations of core/interesting state with the ability
to link identifiers between processes.

### Interesting Roots ###

ContentParent:
* nsDataHashtable<nsStringHashKey, ContentParent*>* sAppContentParents;
* nsTArray<ContentParent*>* sNonAppContentParents;
* nsTArray<ContentParent*>* sPrivateContent;
* StaticAutoPtr<LinkedList<ContentParent> > sContentParents;
`p *('mozilla::dom::ContentParent'::sContentParents.mRawPtr)`

`p *(ContentParent *)mozilla::dom::ContentParent::sContentParents.mRawPtr->sentinel->mNext`

`p ((ContentParent *)mozilla::dom::ContentParent::sContentParents.mRawPtr->sentinel->mNext)->mManagedPPresentationParent`

`p *(('mozilla::dom::ContentParent' *)'mozilla::dom::ContentParent'::sContentParents.mRawPtr->sentinel->mNext)`

`p (('mozilla::dom::ContentParent' *)'mozilla::dom::ContentParent'::sContentParents.mRawPtr->sentinel->mNext)->mManagedPScreenManagerParent`


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
