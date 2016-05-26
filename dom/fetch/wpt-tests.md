* api
  * basic
  * cors
  * credentials
  * headers
  * policies
    * csp-*
    * referrer-*
  * redirect
    * redirect-count: test that up to 20 redirects is fine, 21 fails; all over
      varying redirect status codes
    * redirect-location: test follow/manual mode for valid/invalid/data-url
      location responses over varying redirect status codes
    * redirect-method: verifies POST becomes GET in 301/302/303 cases and
      methods remain intact otherwise.
    * redirect-mode: tests error/follow/manual modes over varying status codes.
  * request
    * request-cache: 
  * response
