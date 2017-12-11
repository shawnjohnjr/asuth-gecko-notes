### Part 1: Dump

Notable:
- Introduces DOMPreferences.{h,cpp}
  - PREF define, which provides cache-on-first-use. AddAtomicBoolVarCache logic.
    - The Preferences API is not actually safe off-the-main-thread, which means
      this logic is dangerous as implemented.  (HaveExistingCacheFor helps
      express this, since the Preferences API isn't great about asserting about
      thread safety, yet... :njn's efforts probably fix a lot of this.)
      - This is largely addressed in Part 23 which brings
        DOMPreferences::Initialize into existence.
- Dump-wise:
  - The ifdef-based conditional nsContentUtils::DOMWindowDumpEnabled() behavior
    that returns true always on DEBUG (or if `MOZ_ENABLE_JS_DUMP`) is propagated
    to DOMPreferences.cpp so that true is still returned in those cases.
