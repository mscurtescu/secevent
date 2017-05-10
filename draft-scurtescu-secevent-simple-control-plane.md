---
title: Simple Control Plane for SET Delivery
abbrev: simple-control-plane
docname: draft-scurtescu-secevent-simple-control-plane-00
date: 2017-05-10
category: info

area: Security
workgroup: secevent
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: M. Scurtescu
    name: Marius Scurtescu
    organization: Google
    email: mscurtescu@google.com
 -
    ins: A. Richard
    name: Annabelle Richard
    organization: Amazon
    email: richanna@amazon.com
    
--- abstract

SET (Secutity Event Token) delivery requires a control plane to get or update
the stream status, to add or remove subjects and to trigger verification
events. This specification defines a plain REST API that implements a basic
control plane. The main purpose of this specification is to offer a comparison
to a similar SCIM REST API.

--- middle

Introduction {#intro}
============

Control Plane Operations {#operations}
========================

Stream Status {#status}
-------------

Add Subject {#add}
-----------

Remove Subject {#remove}
--------------

Trigger Verify Event {#verify}
--------------------

--- back

Examples {#examples}
========


