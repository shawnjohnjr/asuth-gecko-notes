### test_notificationclick_focus.html ###
It's a general mochitest.

General sequence of events:
* set preferences enabling SW's, SW testing, notifications from workers,
  notification prompt testing, disabling open click delay(?).
* registers the mock notifications helper from MockServices.js
* creates a test frame which waits for the SW to be ready, triggering a
  notification at that point.  This would fail if there were an existing
  registration, but there shouldn't be one.
* registers notificationclick_focus.js SW with a scope of the
  notificationclick_focus.html page
