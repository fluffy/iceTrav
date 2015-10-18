---
title: Firewall Traversal for WebRTC
abbrev: WebRTC Firewall
docname: draft-jennings-behave-rtcweb-firewall-03
date: 2015-10-18
category: info

ipr: trust200902
area: Art
workgroup: rtcweb
keyword: Internet-Draft


stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: P. Patel
    name: Pradeep Patel
    organization: Cisco
    email: pradpate@cisco.com
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
 -
    ins: D. Wing
    name: Dan Wing
    organization: Cisco 
    email: dwing@cisco.com


normative:
  RFC2119:
  RFC5389:
  RFC3629:
 
informative:
  I-D.ietf-avtcore-rfc5764-mux-fixes:
  I-D.ietf-rtcweb-overview:
  I-D.ietf-rtcweb-stun-consent-freshness:
  RFC4787:
  RFC6066:

--- abstract

Traversal of RTP through corporate firewalls has traditionally been
complex, requiring the deployment of Session Border Controllers (SBCs)
or wide open pinholes. This draft proposes a simple technique that
allows WebRTC based RTP traffic to traverse firewalls without complex
firewall configuration and without deployment of SBCs or other
middleboxes.

--- middle


Problem Statement
=============

WebRTC {{I-D.ietf-rtcweb-overview}} based voice and video
communications systems are becoming far more common inside
enterprises, which often need voice and video media to traverse the
enterprise firewall. This can happen when a device inside the firewall
such as a web browser or phone is exchanging media with a conference
bridge or gateway outside the firewall, or it can happen when a device
inside the firewall is talking to a device in another enterprise or
behind a different firewall.

This problem is not unique to WebRTC media of course. It is common
practice for enterprise administrators to block outbound UDP through the
corporate firewall. This is done for several reasons:

1. The lack of any kind of return messages means that there is no way to
know that the recipient of the UDP traffic really wants it. Infected
computers within the enterprise could utilize UDP as the source of a
DDoS attack. If the firewall permitted such outbound traffic, the
enterprise could in effect be a contributing source to such an
attack. By blocking UDP, the enterprise IT admin ensures that this
cannot happen - at least not to external targets.

2. There have been prior attacks that have utilized UDP as a command and
control channel for orchestrating DDoS attacks. At the time, UDP had
little usage within enterprises (most VoIP was internal to the
enterprise when it existed at all). Consequently, infosec departments
have deemed it safer to block UDP outright in order to prevent such
further incidents.

3. Many IT administrators enable various packet inspection operations on
traffic flowing through the firewall. High volume UDP traffic - such as
voice or video - can be costly to inspect. As such, in cases where there
is a need for traversal of such traffic, IT has preferred to deploy an
SBC that, in essence, verifies that the traffic is VoIP and authorizes
its egress. The IT administrator then enables traffic to/from the SBC
through the firewall. In other words, VoIP authorization is delegated to
an outsourced SBC.

As more and more IP communications services move to the cloud, there is
an increased need for VoIP traffic to traverse the enterprise
firewall. At the same time, the entire point of a cloud service is that
it does not require the deployment of on premises infrastructure, making
SBC-based solutions less desirable. An alternative solution that has
been historically used is to enable outbound UDP in the firewall to
specific IP addresses, corresponding to the external service (TURN
servers or conference servers) that the enterprise wishes to
authorize. With more applications running on virtual machines within
cloud compute platforms like Amazon EC2, IP addresses are decreasingly
usable as identifiers for a service. VMs running TURN servers or
conferencing servers may be established and torn down by the day, hour
or even minute, with continuously changing IP addresses. Given the
multitenant nature of such providers, IT departments are unwilling to
whitelist the IP addresses for the entire block used by such providers.

Consequently, there is a growing need for solutions that allow VoIP
traversal through the corporate firewall that alleviate the concerns
above. This issue is further exacerbated by the growing adoption of
WebRTC by enterprise applications, which provide a ready source of RTP
traffic which often needs to traverse the firewall.

Solution Requirements
================

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
enterprise which terminate media on servers, such as conference
servers, voicemail servers, and so on.

REQ-6: The solution must not require decryption of either signaling
or media traffic at the firewall or at any other intermediary

REQ-7: The solution must allow the IT department to easily make policy
decisions about which applications are allowed, or not allowed, to
traverse the firewall

REQ-8: The solution must not require inspection of every single packet
that traverses the firewall

REQ-9: The solution must provide a minimum level of proof that the
traffic is WebRTC media or data and not something else

REQ-10: The solution must work with WebRTC traffic. Note that solving
this for non-WebRTC is a non-requirement.


Solution Overview
============

Many of the reasons for blocking UDP at the corporate firewall have
their origins in the lack of a three-way handshake for UDP
traffic. TCP's three-way handshake ensures that the receiving party of
the connection desires the traffic. Similarly, HTTP traffic easily
traverses the firewall since it provides application identification
information in the URL.

Consequently, the solution proposed here relies on the ICE connectivity
checks, which provide a similar handshake and ensure consent of the
remote party.

The firewall looks for an initial STUN transaction to a STUN server to
learn which application is using the port (based on the STUN HOST
attribute {{sec-stun_host}}). Next the firewall watches the outbound ICE
connectivity check on that port and allows inbound ICE connectivity
checks that are going to the same location that sent the outbound
request and that have the correct random ufrag value that was created by
the client inside the firewall.  After a successful ICE connectivity
check, the firewall allows other media to flow on the same 5 tuple that
had the successful ICE connectivity check.  Timers are used to removed
the various pinholes created.

In addition, the initial outbound STUN packets can contain the STUN HOST
attribute which the firewall can use to make an authorization decision
on the application.

The end result is a system where:

* STUN packets are only allowed “in” if they know the crypto random
  username generated by a client inside the firewall

* STUN packers going “out” can be restricted by policy based on the
  hostname of the STUN server they are using

* Non STUN packets are only allowed “in” if they match a 5 tuple that a
  client inside the firewall sent a packet too

* Non STUN packets are only allowed “out” if the destination they are
  sending to did a stun consent handshake


Firewall Processing
==============

The firewall processing is broken into four stages: recognizing STUN
packets, mapping to an application, making a policy decision as to
whether each STUN packet should trigger a pinhole to be created, and
managing the lifetime of any pinholes that are created.

Terminology
----------

The key  words defined in {{RFC2119}} are used in this specification.

The term 3-tuple is used to refer to IP address, protocol (which is
always UDP), and port that the firewall sees as the address of the
client inside the firewall.

The term 4-tuple is used to refer to 3-tuple plus the ice ufrag that
was send in the STUN request message for the client inside the
firewall.

The term 5-tuple is used to refer to the 3-tuple plus the IP address
and port of the device outside the firewall.

When matching a ufrag, if it is a STUN request that came from outside
the firewall, the two halves of the username on either side of the ":"
need to be swapped before matching.


Recognizing STUN packets
------------------------

STUN messages all have a magic cookie value of 0x2112A442 in the 4th to
8th byte. This can be used to quickly filter nearly all UDP packets that
are not STUN packets. Many firewalls are capable of doing this in
hardware. STUN supports an optional FINGERPRINT attribute that provides
a 32 bit CRC over the message.

Option A: Firewalls SHOULD look at outbound UDP packets and if they have
the correct magic cookie they can classify them as STUN packets.

Option B: The firewall looks for any outgoing STUN requests to the STUN
port (3478). When it finds one, it stores the 3 tuple of the source
address port and protocol=UDP and for the next 30 seconds checks any
packets from this 3 tuple to see if they are ICE connectivity checks. 

Open Issues:

* decide between option A and B. A requires looking at all UDP packets
  but will likely work better than B. Most firewalls look at all TCP
  packets so probably not too big of a deal.

* MAY, MUST, MUST NOT look at FINGERPRINT - what do we want here. If we
  put MAY or MUST, then browsers MUST include this. If browsers are not
  required to provide this then I think we are more in the MUST NOT
  category. If we do not use the fingerprint, there will be some small
  number of false positives.

* Should do the analysis to see what harm comes of treating random
  packets as STUN packets.

* CJ Proposal: Browsers MUST send the fingerprint when sending STUN
  messages to STUN server but MAY use it when doing ICE connectivity
  checks. This is to help save bandwidth as Justin Uberti was suggesting
  that change from approximately 50kbps to 100 kbps for ICE makes big
  difference for mobile devices.


Application Mapping
-----------------

The STUN HOST attribute {{sec-stun_host}} carries the fully qualified
domain name of STUN server that is being contacted in the STUN
requests. So for example, if a browser was on a page such as example.com
and that page used the WebRTC calls to set up a connection to
stun.example.com, the STUN request's HOST attribute would have the value
stun.example.com. Similarly when contacting a STUN or TURN server over
TLS or DTLS, the TLS SNI {{RFC6066}} value provides the name of the
host. For systems that provide a unique STUN server name for each
application, this allows the firewall to map the stun port to the
application using it and use that for logging and policy decisions.

Once the Firewall receives as STUN packet from the inside to the outside
on a new 3-tuple. It MUST create an internal record to track any
additional traffic on this 3-tuple. If the STUN packets contains an HOST
attribute then the value it contains is saved in this record and
referred to as the applications name. Firewall might wish to put the
application name in the log files for this 3-tuple.

It is important to realize that any application inside the firewall can
lie about the value of the HOST attribute. However, a web browser that
is trusted will not allow the Javascript running in the web browser to
lie about the value of the HOST.


Policy decision
--------------

Once the firewall has received a STUN packet from inside the firewall,
it needs to decide if the packet is acceptable. For most situations the
firewall SHOULD accept all outbound STUN packets. This is similar to
allowing all outbound TCP flows. Some firewalls may choose to look at
other factors including the outside UDP port and the application name
for this 3-tuple.

In general WebRTC media can be sent on a wide range of UDP ports but the
two ports that are commonly used are the the RTP port (5004) and TURN
port (3478). Some firewalls MAY choose to only allow flows where the
destination port on the outside of the firewall is one of these.

Some firewalls MAY decide to white or blacklist connections based on the
application name.


Creating the pinhole rules
---------------------------

Once a STUN packet is accepted, the firewall MUST create a temporary
rule that causes the firewall to allow any inbound or outbound ICE
messages on this 4-tuple. This pinhole MUST to be valid for at least 5
seconds from the time of creation.

The firewall keeps track of the STUN transaction ID for all STUN
requests messages that traverse the 4 tuple along with the 5 tuple they
were sent on and direction (inbound or outbound). If the firewall sees a
STUN Success binding responses, with the same transaction ID, and on the
same 5 tuple but in the opposite direction as the STUN request, then a
valid ICE connectivity check has happened and the firewall MUST create a
pinhole for this 5 tuple that allows any UDP traffic to flow across that
5 tuple. This pinhole MUST to be valid for at least 30 seconds from the
time of creation.

The firewall continues watching ICE connectivity checks across this
5-tuple as described in the previous paragraph and anytime the a valid
ICE connectivity check happens, this effectively extends the lifetime of
the pinhole by 30 seconds. The procedures in
{{I-D.ietf-rtcweb-stun-consent-freshness}} will ensure that an ICE
connectivity check is done more often than every 30 seconds. This is
designed to make things work with behave compliant NATS and Firewalls as
specified in {{RFC4787}}.


Media vs Data Statistics
----------------------

WebRTC can send audio and video as well as carry a data
channel. Confidential data could leave an enterprise by a video camera
being pointed at a document, but IT departments are often more concerned
about the data channel. It is easy for the firewall to separately track
the amount of RTP media and non-media data for each WebRTC flow. If the
first byte of the UDP message is 23, it is non-media data; if it is in
the range 127 to 192 it is audio or video data. More information about
this can be found in {{I-D.ietf-avtcore-rfc5764-mux-fixes}}. Network
management systems on the firewall can track these two separately which
can help identify unusual usage.


WebRTC Browsers
============

This specification would require browsers to include the FINGERPRINT and
HOST attributes in STUN requests to a STUN server used to gather
candidates for this to work correctly. Note they would not need to be
included in STUN requests sent peer to peer or sent to ensure media
consent.
 
Open Issue: how much randomness for ICE ufrag

* ICE mandates at least 24 bits of randomness but we could require the
  browsers produce 64 bits of randomness?

Open Issue: Does adding the HOST reduce user privacy?

* Consider the following case. The user goes to https://facebook.com and
  initiates a call with another Facebook user using facebook.com as the
  name of the STUN server. The domain facebook.com will appear
  (unencrypted) in the STUN packets sent from the browser to Facebook's
  TURN server. Anyone along the network path could tell that the user is
  using Facebook's TURN server. However, when the original TLS
  connection for the HTTP was made, the Server Name Indication (SNI) in
  the TLS of the HTTPS connection also revealed facebook.com, largely
  for the same reasons - so that the firewall would be able to see which
  applications are using the network.

Open Issue: Would only including HOST when it matched the HTTP ORIGIN
improve privacy?

* We could make this so that when used with WebRTC browsers, the HOST is
  only included in the STUN check when the name of the STUN servers
  matches the HTTP ORIGIN of the web page initiating the STUN
  request. It is not clear if this would improve privacy or not.



STUN HOST attribute {#sec-stun_host}
====================

This specification defines a new STUN attribute called HOST and uses the
syntax defined in Section 15 of {{RFC5389}}. This attribute is of type
comprehension-optional. The value of the HOST attribute is a variable
length value. It MUST contain a UTF-8 {{RFC3629}} encoded sequence of
characters.

The HOST attribute identifies the fully qualified domain name of the
application provider that is serving the WebRTC application and also
operating the STUN server. The WebRTC EndPoint MUST include this
attribute as part of ICE candidate gathering checks. There MUST be only
one HOST attribute in a given ICE connectivity check and MUST be
included before the MESSAGE-INTEGRITY attribute to ensure its integrity.


Deployment Advice
=============

WebRTC Servers
--------------

WebRTC media servers and TURN servers with public IP address(es) that
can receive incoming packets from anywhere on the Internet are suggested
to listen for UDP on ports 5004 for RTP media servers and 3478 for TURN
servers. UDP destined for port 53 or 123 if often allowed by firewalls
that otherwise block UDP.


Firewall Admins
-------------

Often the approach has been to lock down everything, so that all UDP is
blocked. This simply causes applications to do things like embed the
data in normal looking HTTP or HTTPS requests. Malware and viruses use
similar approaches. Just turning off all UDP results in a poor user
experience some of the time, which results in users moving to
applications and devices outside the firewall. The IT department loses
the visibility into what is going on and can no longer protect its users
when their computers become compromised. Allowing things that users want
to use to work and monitoring them to detect when things have gone wrong
is very valuable.


Design Consideration
==============

Why not just use TCP?
------------------

TODO

IANA Considerations
==============

IANA is requested to add the HOST attributes to the STUN attribute
registry. The values for HOST is to be allocated from the expert review
comprehension-optional range of (0xC000 - 0xFFFF).

| Value | Name | Reference |
|-----|-----|--------|
| TBD  | HOST | RFCXXXX  |

This specification defines the HOST attribute in {{sec-stun_host}}.

Security Considerations 
=================


Enterprises have a range of concerns around WebRTC traffic traversal of
the firewall. The major concerns that are raised include:

1. Unlike TCP, UDP does not have a connection where a device inside the
   firewall has confirmed that it wants to talk to the thing outside.

2. Incoming UDP pinholes allow out of band packets to be spoofed into
  connecting as there is no equivalent of a TCP sequence number to
  check.

3. UDP has been used by malware command and control protocols so we
   block it.

4. We do not want enable ways for data to be exfiltrated outside the
   firewall with no monitoring.

5. An encrypted data channel in WebRTC can be used to bring malware into
   the company.

6. An encrypted media or data channel in WebRTC can be used as a command
   and control channel for malware inside the firewall.

7. An encrypted data channel in WebRTC can be used by an outside
   attacker to exfiltrate private files from inside the firewall.

TODO - Describe to what degree theses are addressed. Be clear about
attacks due to Javascript inside the firewall and attacks due to
executables inside the firewall.


Privacy
------

Unlike previous version of this draft, we think that using HOST instead
of ORIGIN minimizes any privacy concerns. The HOST is already known to
the operator of the STUN server as they run it. It often contains
exactly the same information as the existing STUN REALM attribute. It
has roughly the same information as the TLS SNI {{RFC6066}} when STUN is
run over DTLS.



Alternate Approaches 
===============


SDN Control of Firewall
--------------------

An alternative ways of solving this problem is for the Web Application
running in the browser to inform the web site what ports and IP
addresses it is using then the web site to contact the appropriate SDN
controller and request the SDN controller tell the appropriate firewall
what pinholes to open.  This can be made to work in some deployments but
not all as it is often not clear how to find the correct SDN controller
or set up a relationship such that the SDN controller trusts a website
outside the firewall wall enough to let it tell the controller to open
wholes in the Firewall.

SDN based approaches should be pursued as well as this approach as they
compliment each other.


Any Cast White List
---------------

Deploying media or TURN servers on a single any-cast IP address also
makes it easier for firewall administrators to whitelist the
address. Concerns have been raised that two packets sent from the same
host to a given any-cast address may get delivered to different servers.
This is certainly possible in theory but in practice it does not seem be
happen in limited experiments done so far.


Acknowledgements
=============

Many thanks to review from Shaun Cooley, Teh Cheng, and Alissa Cooper.

The definition of HOST STUN attribute was motivated by discussion around
the draft-ietf-tram-stun-origin document and we want to thanks Alan
Johnston, Justin Uberti, John Yoakum, and Kundan Singh.

