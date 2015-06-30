---
title: Firewall Traversal for WebRTC
abbrev: WebRTC Firewall
docname: draft-jennings-rtcweb-firewall-latest
date: 2015-07-30
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
    organization: CISCO
    email: fluffy@iii.ca

normative:
  RFC2119:

informative:
  I-D.ietf-rtcweb-overview:


--- abstract

This draft propose a way for enterprise firewalls to handle WebRTC
media traffic. 

--- middle


Overview 
=========

WebRTC based voice and video communications systems are becoming far
more commone and they often need voice and video media to traverse the
enterprise firewall. This may because a device inside the firewall is
exchanging media with a confernce bridge or gateway outside the
firewall or it may be because a device inside the firewall is talking
to a device in another enterprise or behind a different firewall.

Firewalls admistrators have often been unwilling to open UDP due to
some concerns evne thoguht TCP may be allowed due to it full handshake
that verfies that a device inside the firewall really does want to
communicate with the outside device. WebRTC media has many of the same
characteristics as TCP and a firewall can use thoses to acheive
security compariable to TCP connection intiated inside the firewall.

WebRTC media connections all start with STUN consent checks. Thoses
are used 



Firewall Processing
==============

recognizing stun packets

sugested media ports

tracking the ICE consent messages

creating the pin hold rules

tracking media vs data

Firewall Proteciton
============

blocking media over HTTPS


Deployment Advice
==============


WebRTC Servers
--------------

Run on port XXX

Provide a UDP turn server




Design Consideration
==============

Why not just use TCP?
------------------


Security Concerns 
============


Enerprises have a range of concerns around WebRTC traffic traversal of
the firewall. The major concers that are reaised include:

1. Unlike TCP, UDP does not have a connection where a device inside
   the firewall has confirmed that it wants to talk to the thing
   outside.

2. Incoming UPD pinholes allow out of band packets to be spoofed into
   the connectiong as there is no equivelent of TCP sequence number to
   check

3. UDP has been used by malware command and controll protcols so we
block it.

4. Do not want enable ways for data to be exfilterated outside the
firewall with no monitoring

5. An encrypted data channel in WebRTC can be used to bring malware
into the company

6. An encrypted data channel in WebRTC can be used by an outside
attacker to upload private files from inside the firewall


