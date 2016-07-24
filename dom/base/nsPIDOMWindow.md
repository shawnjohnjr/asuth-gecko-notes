Provides specific types nsPIDOMWindowInner and nsPIDOMWindowInner to reify the
semantic concepts of outer and inner window into the type system, even though
nsGlobalWindow is the concrete type of everything.

Somewhat analogous to the type distinction between WorkerPrivate (for use on
the worker thread) and WorkerPrivateParent (for parent-thread stuff).

General inner/outer docs with very useful diagrams:
https://developer.mozilla.org/en-US/docs/Inner_and_outer_windows

### Type gunk ###

nsGlobalWindow:
  * isa nsPIDOMWindow<nsISupports>
nsPIDOMWindow<T>
* nsPIDOMWindowInner : public nsPIDOMWindow<mozIDOMWindow>
  * mozIDOMWindow : nsISupports {}
* nsPIDOMWindowOuter : public nsPIDOMWindow<mozIDOMWindowProxy>
  * mozIDOMWindowProxy : nsISupports {}
Related:
* interface nsIDOMWindow : nsISupports {};
* interface nsIDOMWindowInternal : nsIDOMWindow {};

Split bug:
https://bugzilla.mozilla.org/show_bug.cgi?id=1241764
