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
 -
    ins: S. Nandakumar 
    name: Suhas Nandakumar 
    organization: Cisco 
    email: snandaku@cisco.com 
 -
    ins: J. Rosenberg 
    name: Jonathan Rosenberg
    organization: Cisco 
    email: jdrosen@cisco.com


normative:
  I-D.ietf-tram-stun-origin:
  I-D.ietf-avtcore-rfc5764-mux-fixes:
 
informative:
  I-D.ietf-rtcweb-overview:
  I-D.ietf-rtcweb-stun-consent-freshness:
  I-D.reddy-rtcweb-stun-auth-fw-traversal:

--- abstract

Traversal of RTP through corporate firewalls has traditionally been
complex, requiring the deployment of Session Border Controllers (SBCs)
or wide open pinholes. This draft proposes a simple technique that
allows WebRTC based RTP traffic to traverse firewalls without complex
firewall configuration and without deployment of SBCs or other
middleboxes. 

--- middle


Problem Statement
=========

WebRTC {{I-D.ietf-rtcweb-overview}} based voice and video
communications systems are becoming far more common inside of
enterprises and they often
need voice and video media to traverse the enterprise firewall. This
can happen when a device inside the firewall such as a web browser
or phone is exchanging media with a conference bridge or gateway
outside the firewall or it can happen when a device inside the
firewall is talking to a device in another enterprise or behind a
different firewall.

This problem is not unique to WebRTC media of course. It is common
practice for enterprise administrators to block outbound UDP through
the corporate firewall. This is for several reasons:

1. The lack of any kind of return messages means that there is no way
to know that the recipient of the UDP traffic really wants
it. Infected computers within the enterprise could utilize UDP as the
source of a DDoS attack. If the firewall permitted such outbound
traffic, the enterprise could in effect be a contributing source to
such an attack. By blocking UDP, the enterprise IT admin assures that
this cannot happen - at least not to external targets.

2. There have been prior attacks that have utilized UDP as a
command and control channel for orchestrating DDoS attacks. At the
time, UDP had little usage within the enterprise (most VoIP was
internal to the enterprise when it existed at all). Consequently,
infosec departments deemed it safer to block UDP outright in order to
prevent such a further incident.

3. Many IT administrators enable various packet inspection operations
on traffic flowing through the firewall. High volume UDP traffic -
such as voice or video - can
be costly to inspect. As such, in cases where there is a need for
traversal of such traffic, IT has preferred to deploy an SBC that, in
essence, verifies that the traffic is VoIP and authorizes its
egress. The IT administrator then enables traffic to/from the SBC
through the firewall. In otherwords, VoIP authorization is delegated
to an outsourced SBC. 

As more and more IP communications services move to cloud, there is an
increased need for VoIP traffic to traverse the enterprise
firewall. At the same time, the entire point of a cloud service is
that it does not require deployment of premises
infrastructure, making SBC-based solutions less desirable. An
alternative solution that has been historically used is to enable
outbound UDP in the firewall to specific IP addresses, corresponding
to the external service (TURN servers or conference servers) that the
enteprise wishes to authorize. With more applications running on
virtual machines within cloud compute platforms like Amazon EC2, IP
addresses are decreasingly usable as identifiers for a
service. VMs running TURN servers or conferencing servers may be
established and torn down by the day, hour or even minute, with
continuously changing IP addresses. Given the multitenant nature of
such providers, IT departments are unwilling to whitelist the IP
addresses for the entire block used by such providers.


Consequently, there is a growing need for solutions
that allow VoIP traversal through the corporate firewall that
alleviate the concerns above. This issue is further exacerbated by the
growing adoption of WebRTC by enterprise applications, which provide a
ready source of RTP traffic which often needs to traverse the
firewall.

Solution Requirements
=====================

We believe the solution must meet the following requirements:

REQ-1: The solution must enable traversal of real-time media without
requiring deployment of additional media intermediaries on premise
(e.g., no SBC required)

REQ-2: The solution must not require the whitelisting of specific
external IP addresses

REQ-3: The solution must enable the enterprise to be sure that the
receiving party of the traffic desires the traffic

REQ-4: The solution must work with P2P calls between users in
different enterprises without requiring a TURN server

REQ-5: The solution must work with cloud services external to the
enteprise which terminate media on servers, such as conference
servers, voicemail servers, recording servers, and so on.

REQ-6: The solution must not require decryption of either signaling or
media traffic at the firewall or at any other intermediary

REQ-7: The solution must allow the IT department to easily make policy
decisions about which applications are allowed, or not allowed, to
traverse the firewall

REQ-8: The solution must not require DPI of ever single UDP packet
that traverses the firewall

REQ-9: The solution must provide a minimum level of proof that the
traffic is RTP and not something else

REQ-10: The solution must work with WebRTC traffic. Note that solving
this for non-WebRTC is a non-requirement.


Solution Overview
=================

Many of the reasons for blocking UDP at the corporate firewall have
their origins in the lack of a three way handshake for UDP
traffic. TCPs three-way handshake ensures that the receiving part of
the connection desires the traffic. Similarly, HTTP traffic easily
traverses the firewall since it provides application identification
information in the URL.

Consequently, the solution proposed here relies on the ICE
connectivity checks, which provide a similar handshake and ensure
consent of the remote party. When a firewall sees an outbound UDP
packet on a 5-tuple which is not yet authorized, it begins looking for
STUN packets (as identified by the STUN magic cookie). Any outbound
packet that is not a STUN packet is discarded. Once an outbound STUN
packet is identified, the 5-tuple is put in a pending state, and the
firewall begins looking for STUN packets within inbound UDP
packets. When it sees one, it matches the transaction IDs to ensure
that they are correlated. Once matched, the 5-tuple is placed into an
authorized state, and UDP traffic is allowed to freely traverse.

In addition, the initial outbound STUN packets contain the STUN ORIGIN
field which the firewall can use to make an authorization decision on
the application. 


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


Policy decision
--------------

Once the firewall has received a STUN packet from inside the firewall,
it needs to decide if wants to accept that or not. For most situations
the firewall SHOULD accept all outbound STUN packets. This is similar
to allow all outbound TCP flows. Some firewalls may choose to look at
other factors including the outside UDP port and the ORIGIN attribute
in the STUN packet.

In general WebRTC media can be sent on a wide range of UDP ports but
the two ports that are commonly used are the the RTP port (5004) and
TURN port (3478). Some firewalls MAY choose to only allow flows where
the destination port on the outside of the firewall is one of theses.

The STUN ORIGIN attributes {{I-D.ietf-tram-stun-origin}} carries the
origin of the web page that caused the various STUN requests. So for
example, if a browser was on a page such at example.com and that page
used the WebRTC calls to set up a connection, the STUN request's ORIGIN
attribute would include example.com. This allows the firewall to see the
web applications (in this case, example.com) that is requesting the
pin hole be opened. The firewall MAY have a white list or black list
for domain in STUN ORIGIN.


Creating the pin hole rules
---------------------------

Once a STUN packets it accepted, the firewall MUST create a temporary
rules that allows incoming and outgoing packets for that 5 tuple for
at least 5 seconds. If in that 5 seconds, a response is received to
the STUN request, the lifetime of the rule must be extended to at
least 30 seconds from last accepted STUN packet from inside the
firewall. Once extended to 30 seconds, any additional UDP
packets from inside the firewall MUST extend the lifetime of the rule
by at least 30 seconds from the time of that packet was received. The
procedures in {{I-D.ietf-rtcweb-stun-consent-freshness}} will ensure
that an outbound packet is sent at least every 30 seconds. 


Tracking media vs data
----------------------

WebRTC can send both audio and video as well as data
channel. Confidential data could leave an enterprise by a video camera
being pointed out a document but IT departments are often more
concerned about the data channel. It is easy for the firewall to
separately track the amount of RTP media and non media data for each
WebRTC flow. By looking at the first byte of the UDP message,
if it is 23 it is non media data while if it is in the range 127 to 192 
it is audio or video data. More information about this can be found in
{{I-D.ietf-avtcore-rfc5764-mux-fixes}}. Network management systems on 
the firewall can track these two separately which can help identify 
unusual usage.


WebRTC Browsers
===============

This specification would require browsers to include the FINGERPRINT
and ORIGIN attributes in STUN for this to work correctly.

Open Issue: Does add the ORIGIN reduce user privacy. Consider the
following case, the user goes to https://facebook.com and initiated a
call with another facebook user. The domain facebook.com will appear
(unencrypted) in the STUN packets sent from the browser to the
faceboks TURN server. Anyone along the network path could tell that
the user is using facebook's TURN server. However, when the original
TLS connection for the HTTP was made, the Server Name Indication (SNI)
in the TLS of the HTTPS connection also revealed facebook.com and this
was done for largely the same reasons, so the Firewall would be able
to see which applications were using the network.


Blocking Media Hiding in HTTP
============

The IETF is designing systems to send interactive audio and video such
that it looks like HTTPS and HTTP to the proxies and firewalls.  The
reasons for doing this is that sometimes the proxies and firewalls
allow this to work while the mechanisms and channels designed for
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
receive incoming packets from anywhere on the Internet are suggested to
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
=================


Enterprises have a range of concerns around WebRTC traffic traversal
of the firewall. The major concerns that are raised include:

1. Unlike TCP, UDP does not have a connection where a device inside
   the firewall has confirmed that it wants to talk to the thing
   outside.

2. Incoming UDP pinholes allow out of band packets to be spoofed into
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


Alternate Approaches 
=====================

Firewall Auth Tokens
------------------

{{I-D.reddy-rtcweb-stun-auth-fw-traversal}} attempts to solve a similar
problem by proposing a new comprehension-optional FW-FLOWDATA STUN attribute
as part of ICE Connectivity checks enabling the firewall to permit outgoing
UDP flows across the firewall. FW-FLOWDATA STUN provides necessary
information, such as lifetime, candidate information , enabling a firewall
to apply the required policy rules. However, {{I-D.reddy-rtcweb-stun-auth-fw-traversal}}
requires establishing shared keys across the firewall(s) and the WebRTC server 
for successfully verifying the authenticity of the FW-FLOWDATA information.
In summary, we believe {{I-D.reddy-rtcweb-stun-auth-fw-traversal}} to have 
following short-comings

1. Requiring a tight coupling between the application server (WebRTC server) and
firewall(s)

2. Requiring additional efforts for Firewall Admins within a enterprise to distribute 
and maintain the shared authentication keys needed to generate authentication
tag for the FW-FLOWDATA attribute.

3. {{I-D.reddy-rtcweb-stun-auth-fw-traversal}} doesn't apply for 
distributing keys across firewalls in different administrative domains.

Any Cast Whitelist
---------------

Deploying media or TURN servers on a single any-cast IP address also 
makes it easier for firewall administrators to whitelist the 
address. Concerns have been raised that two packets sent from the same 
host to a given any-cast address may get delivered to different 
servers.  This is certainly possible in theory but in practice it does 
not seem be happen in limited experiments done so far. 


Acknowledgements
================

Many thanks to ...



