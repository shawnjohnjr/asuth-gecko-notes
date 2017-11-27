### Ownership ###

PBackgroundSDBConnectionChild is explicitly deleted, not refcounted, per
BackgroundChildImpl.cpp.
