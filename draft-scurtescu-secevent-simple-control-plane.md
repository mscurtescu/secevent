---
title: Simple Control Plane for SET Delivery
abbrev: simple-control-plane
docname: draft-scurtescu-secevent-simple-control-plane-00
date: 2017-05-10
category: info
ipr: trust200902

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
    ins: A. Backman
    name: Annabelle Backman
    organization: Amazon
    email: richanna@amazon.com
    
normative:
 RFC7519:

--- abstract

SET (Secutity Event Token) delivery requires a control plane to get or update
the stream status, to add or remove subjects and to trigger verification
events. This specification defines a plain REST API that implements a basic
control plane. The main purpose of this specification is to offer a comparison
to a similar SCIM REST API.

--- middle

Introduction {#intro}
============

Notational Conventions {#conv}
======================
The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in {{!RFC2119}}.

Control Plane Resources {#resources}
=======================
Event receivers interact with a stream's control plane through
RESTful calls to resources under the stream's stream control
endpoint.


Stream {#stream}
----------------
A stream represents a configured relationship between a single
event transmitter and a single event receiver, and describes which
events the transmitter may transmit to the receiver and how the
receiver may receive those events. Transmitters MAY support multiple
streams for the same receiver. Transmitters MAY assign distinct URLs
for different streams, and/or provide a single URL for all streams
(e.g. if the transmitter can identify the stream from the credentials
used to authenticate the request).

{: vspace="0"}
aud
: A string containing an audience claim as defined in [JSON Web Token
  (JWT)](#RFC7519)
  that identifies the event receiver for the stream.

events
: Optional. An array of URIs identifying the set of events which the
  transmitter may transmit over this stream. If omitted, transmitters
  SHOULD make this set available to the receiver via some other means
  (e.g. publishing it in online documentation).

transmission_method
: A string indicating the mechanism by which the transmitter will
  transmit events to the receiver. Allowed values are:
  
  {: vspace="0"}
  HTTP_PUSH
  : The transmitter will send events to the receiver via
    HTTP POST requests to the stream's http_push_url.

  HTTP_PULL
  : The receiver will periodically make HTTP GET requests to
    the stream's http_pull_url to get new events.
  
  <<TODO: (richanna) These should reference the distribution spec.>>\\
  <<TODO: (richanna) Should these be UR\[ILN]s instead of strings I
  made up?>>

http_pull_url
: Optional unless transmission_method is HTTP_PULL. The URL to which
  the receiver will make requests in order to get events.

http_push_url
: Optional unless transmission_method is HTTP_PUSH. The URL to which
  the transmitter will send events when transmitting events to the
  receiver. 

### Reading a Stream 
An event receiver gets the current configuration of a stream by making
an HTTP GET request to the stream resource. On receiving a valid request
the event transmitter responds with a 200 OK response containing a JSON
representation of the stream's configuration in the body.

The following is a non-normative example request to read a stream's
configuration:

~~~
GET /streams/risc HTTP/1.1
Host: transmitter.example.com
Authorization: Bearer eyJ0b2tlbiI6ImV4YW1wbGUifQo=
~~~

The following is a non-normative example response:

~~~
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store
Pragma: no-cache

{
  "aud": "http://www.example.com",
  "transmission_method": "HTTP_PUSH",
  "http_push_url": "https://receiver.example.com/event_receiver/risc",
  "events": [
    "https://schemas.openid.net/risc/event-type/account-at-risk",
    "https://schemas.openid.net/risc/event-type/account-deleted",
    "https://schemas.openid.net/risc/event-type/account-locked",
    "https://schemas.openid.net/risc/event-type/account-unlocked",
    "https://schemas.openid.net/risc/event-type/client-credentials-
        revoked",
    "https://schemas.openid.net/risc/event-type/sessions-revoked",
    "https://schemas.openid.net/risc/event-type/tokens-revoked"
  ]
}
~~~

Subject {#subject}
------------------
Each stream has a set of subject subresources which indicates the
subjects about which events may be transmitted over the stream. Event
receivers manipulate this set to signal to the transmitter which
subjects the receiver wishes to receive events about.

<<TODO: Add attributes corresponding to claims?>>

### Adding a Subject
When a receiver wants to signal to a transmitter that they wish to
receive events about a particular subject over a stream, the receiver
makes an HTTP PUT request to the `/subjects` subresource of the stream.
The transmitter MAY choose to ignore the request, for example if the
subject has indicated to the transmitter that they do not want events to
be transmitted to the receiver. On a successful response, the
transmitter responds with a 200 OK response.

The following is a non-normative example request to add a subject to a
stream:

~~~
PUT /streams/risc/subjects HTTP/1.1
Host: transmitter.example.com
Authorization: Bearer eyJ0b2tlbiI6ImV4YW1wbGUifQo=

{
  "iss": "http://transmitter.example.com",
  "sub": "1234567"
}
~~~

The following is a non-normative example response:

~~~
HTTP/1.1 200 OK
Server: transmitter.example.com
Cache-Control: no-store
Pragma: no-cache

~~~

### Removing a Subject
When a receiver wants to signal to a transmitter that the receiver no
longer wishes to receive events about a subject over a stream, the
receiver makes an HTTP POST request to the `/subjects` subresource of
the stream. The transmitter MAY choose to ignore the request, for
example if the subject has indicated to the transmitter that they do not
want events to be transmitted to the receiver. On a successful response,
the transmitter responds with a 204 (No Content) response.

The following is a non-normative example request to remove a subject
from a stream:

~~~
POST /streams/risc/subjects HTTP/1.1
Host: transmitter.example.com
Authorization: Bearer eyJ0b2tlbiI6ImV4YW1wbGUifQo=
Content-Type: application/json

{
  "iss": "http://transmitter.example.com",
  "sub": "1234567",
  "receive_events": false
}
~~~

The following is a non-normative example response to a successful
response:

~~~
HTTP/1.1 204 No Content
Server: transmitter.example.com
Cache-Control: no-store
Pragma: no-cache

~~~

Verification {#verify}
----------------------

### Triggering a Verification Event.

#### Successful Response

--- back

