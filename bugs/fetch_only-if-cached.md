
# TODO #

* Address review feedback:
  * https://bugzilla.mozilla.org/show_bug.cgi?id=1272436#c5
    [x] fixed crazy indentation on my part.
    [ ] address caching of redirects

# Outstanding #

* Caching of Redirects
  * Meta
    * Not covered in the spec changes
    * Example ehsan gave was "redirect from same-origin to cross-origin and back
      to same-origin".
    * Conclusion reached was that redirects should not be cached, which means
      same-origin redirect to anywhere, even same-origin should fail.
  * Actions
    * **Locate or add test that primes cache with redirect, then uses
      only-if-cached and fails when it hits the cache.**
    * **Locate place in the spec that defines this behavior or should define
      it.**
