### Anny feedback
- Formatting of docs more of an issue than the contents.
- Things could get overwhelming, would dig into the existing implementations.
- Checklists would be useful.
  - ex: update bindings.conf

### WebIDL bindings doc walkthrough.
- Paragraph 3 on "Note that if you're adding new interfaces" is a bit confusing.
- Adding WebIDL bindings to a class.  Very dense.
  - For the cycle collection macros... maybe err on the side of "this is what
    you want to do."
  - right, Marcos is saying he wants the stubs to start from.
  - Maybe simplify the checklist to exactly what to do, and then link out to
    a page that provides an explanation of what's going on and a flowchart of
    how to decide what to do, for each step.
  - For now, maybe move this whole section down to the bottom.
- "C++ reflections of WebIDL constructs"
  - It would be great to go through the interface methods one-by-one, explaining
    the additional features it's adding.

- Add separate memory management document, that could cover cycle collection.
