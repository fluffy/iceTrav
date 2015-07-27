
Blocking Media Hiding in HTTP
============

The IETF is designing systems to send interactive audio and video such
that it looks like HTTPS and HTTP to the proxies and firewalls.  The
reasons for doing this is that sometimes the proxies and firewalls
allow this to work when the mechanisms and channels designed for
sending audio and video data have been explicitly disabled by the
firewall administrators. Many firewall administrators feel this
circumvents the policy they are trying to enforce and desire a way to
prevent this. Any scheme for preventing this has some risk of
impacting normal HTTP traffic, so there is a desire to provide
guidance around ways to do that in this draft.

Any HTTP or HTTPS connection that sends more than 10 requests per
second for longer than 10 seconds should be paused for 1 second, and
any HTTP/S requests from that client's IP address in the 1 second
pause time should be buffered or simply dropped. This strategy ensures
there is no impact to clients other than the one exceeding the rate
limit and minimizes the impact to other applications on the device
while still reducing the incentive to try and run calls this way.
