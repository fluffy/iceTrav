---
title: Firewall Traversal for WebRTC
abbrev: WebRTC Firewall
docname: draft-jennings-rtcweb-firewall-latest
date: 2015-07-04
category: info

ipr: trust200902
area: Art
workgroup: rtcweb
keyword: Internet-Draft


stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: C. Jennings
    name: Cullen Jennings
    organization: Cisco
    email: fluffy@iii.ca

normative:
  I-D.ietf-tram-stun-origin:


informative:
  I-D.ietf-rtcweb-overview:


--- abstract

This draft propose a way for enterprise firewalls to handle WebRTC
media traffic. 

--- middle


Overview 
=========

WebRTC {{I-D.ietf-rtcweb-overview}} based voice and video
communications systems are becoming far more common and they often
need voice and video media to traverse the enterprise firewall. This
can happened when a device inside the firewall such as a web browser
or phone is exchanging media with a conference bridge or gateway
outside the firewall or it can happen when a device inside the
firewall is talking to a device in another enterprise or behind a
different firewall.

Firewalls administrators have often been unwilling to open UDP due to
some concerns even thought TCP may be allowed due to it full handshake
that verifies that a device inside the firewall really does want to
communicate with the outside device. WebRTC media has many of the same
characteristics as TCP and a firewall can use those to achieve
security comparable to TCP connection initiated inside the firewall.

WebRTC media connections all start with STUN consent checks. This
draft proposes that the outbound STUN packer will create a short lived
pinhole in the firewall allowing inbound traffic on the same five
tuple if various policy checks are passed. If a STUN packet is
received in reply to that, the lifetime of the pinhole can be upgraded
to 30 seconds and future STUN packers on the same 5 tuple could
further extend the lifetime for 30 seconds from the last STUN packet
sent from inside the firewall.

The draft also describes the types of policy checks that a firewall
may wish to do and how they are performed including white listing
specific web sites that are allowed to use WebRTC communications.



Firewall Processing
==============

The firewall processing is broken into three stages, recognizing STUN
packets, making a policy decisions about if this should be allowed to
open a pinhole, and managing the lifetime of the pin hole.


Recognizing stun packets
------------------------

STUN messages all have a magic cookie value of 0x2112A442 in the 4th
to 8th byte. This can be used to quickly filter nearly all UDP packets
that are not STUN packets and many firewalls are capable of doing this
in ASICs. STUN support an optional FINGERPRINT attribute that provides
a 32 bit CRC over the message.

Firewalls SHOULD look at outbound UDP packets and if they have the
correct magic cookie they may classify them as STUN packets. Firewalls
that which desire less false positives MAY also check the FINGERPRINT
attribute is correct.

TODO - Check that WebRTC is required to send FINGERPRINT.


Policy decision
--------------

Once the firewall has received a STUN packet from inside the firewall,
it needs to decide if wants to accept that or not. For most situations
the firewall SHOULD accept all outbound STUN packets. This is similar
to allow all outbound TCP flows. Some firewalls may choose to look at
other factors including the outside UPD port and the ORIGIN attribute
in the STUN packet.

In general WebRTC media can be sent on a wide range of UDP ports but
the two ports that are commonly used are the the RTP port (5004) and
TURN port (3478). Some firewalls MAY choose to only allow flows where
the destination port on the outside of the firewall is one of theses.

The STUN ORGIN attributes {{I-D.ietf-tram-stun-origin}} caries the
origin of the web page that caused the various STUN requests. So for
example, if a browser was on a page such at example.com and that page
used the WebRTC calls to set up a connection, the stun requests would
origin would include example.com. This allows the firewall to see the
web applications (in this case, example.com) that is requesting the
pin hole be opened. The firewall MAY have a white list or black list
for domain in STUN ORIGIN.


Creating the pin hold rules
---------------------------

Once a STUN packets it accepted, the firewall MUST create a temporary
rules that allows incoming and outgoing packets for that 5 tuple for
at least 5 seconds. If in that 5 seconds, a response is received to
the STUN request, the lifetime of the rule must be extended to at
least 30 seconds from last accepted STUN packet from inside the
firewall. Once extended to 30 seconds, any additional accepted STUN
packets from inside the firewall MUST extend the lifetime of the rule
by at least 30 seconds from the time of that packet was received.

Tracking media vs data
----------------------

WebRTC can send both audio and video as well as data
channel. Confidential data could leave an enterprise by a video camera
bing pointed out a document but IT departments are often more
concerned about the data channel. It is easy for the firewall to
separately track the amount of RTP media and non media data for each
flow. By looking at the first byte of the UDP message, if it is XXXX
it is non media data while if it is in the range XXX to XXX it is
audio or video data. Network management systems on the firewall can
track these two separately which helps identify unusually usage.


WebRTC Browsers
===============

Note: The text here needs to eventually move to other specifications
but it is included here for now to make it easy to see the whole
system in one document.

The WebRTC specification already requires browser to include the
FINGERPRINT and ORIGIN attributes.


Blocking Media Hiding in HTTP
============

The IETF is designing systems to send interactive audio and video such
that it looks like HTTPS and HTTP to the proxies and firewalls.  The
reasons for doing this is that sometimes the proxies and firewalls
allow this to work while the mechanism and channels designed for
sending audio and video data have been explicitly disabled by the
firewall administrators. Many firewall administrators feel this
circumvents the policy they are trying to enforce and desire way to
prevent this. Any scheme for preventing this has some risk of
impacting normal HTTP traffic so there is a desire to provide guidance
around ways to do that here.

Any HTTP or HTTPS connection that sends more than 10 requests per
second for longer than 10 seconds should be paused for 1 second and
any HTTP/S requests from that clients IP address in the 1 second pause
time buffered or simply dropped. This strategy ensure there is no
impact to clients other than the one doing this and minimizes the
impact to other applications on the device while still reducing the
incentive to try and run calls this way.


Deployment Advice
==============


WebRTC Servers
--------------

WebRTC media server and TURN server with public IP address that can
receive incoming packets from anywhere are the internet are suggest to
listen for UDP on ports 53 (DNS), 123(NTP), and 5004 for RTP media
servers and 3478 for TURN servers. UDP destined to port 53 or 123
often is allowed by firewalls that otherwise block UDP.


Firewall Admins
---------------

Often the approach has been to lock down everything that does not have
to absolutely not be locked down and all UDP is blocked. This simply
causes applications to do things like embed the data in normal looking
HTTP or HTTPS requests. Malware and viruses simply use other
approach. Just turning off all UDP results in a crappy user experience
some of the time which results in the users moving to applications and
devices outside the firewall. The IT department looses the visibility
into what is going on and can no longer protect their users when their
computers become compromised. Allowing things that users want to use
to work and monitoring them to detect when things have gone wrong can
be very valuable.


Design Consideration
==============

Why not just use TCP?
------------------

TODO


Security Concerns 
============


Enterprises have a range of concerns around WebRTC traffic traversal
of the firewall. The major concerns that are raised include:

1. Unlike TCP, UDP does not have a connection where a device inside
   the firewall has confirmed that it wants to talk to the thing
   outside.

2. Incoming UPD pinholes allow out of band packets to be spoofed into
   the connecting as there is no equivalent of TCP sequence number to
   check

3. UDP has been used by malware command and control protocols so we
block it.

4. Do not want enable ways for data to be exfiltrated outside the
firewall with no monitoring

5. An encrypted data channel in WebRTC can be used to bring malware
into the company

6. An encrypted data channel in WebRTC can be used by an outside
attacker to upload private files from inside the firewall


TODO 
=====

Ref and explain how fits in with reddy-rtcweb-auth-fw-traversla


Acknowledgements
================

Many thanks to ...



