---
title: Control Plane for SET Delivery
abbrev: set-control-plane
docname: draft-scurtescu-secevent-simple-control-plane-00
date: 2017-05-31
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
  JSON: RFC7159
  RFC7519:
  SET:
    title: Security Event Token (SET)
    target: https://tools.ietf.org/html/draft-ietf-secevent-token-01
  DELIVERY:
    title: SET Token Delivery Using HTTP
    target: https://github.com/independentid/Identity-Events/blob/master/draft-hunt-secevent-delivery.txt

--- abstract

Security Event Token (SET) delivery requires event receivers to indicate
to event transmitters the subjects about which they wish to receive
events, and how they wish to them. This specification defines an HTTP 
API for a basic control plane that event transmitters can implement and
event receivers may use to manage the flow of events from one to the
other.

--- middle

Introduction {#intro}
============
<<TODO>>

Notational Conventions {#conv}
======================
The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in {{!RFC2119}}.

Definitions {#def}
===========
In addition to terms defined in [SET](#SET), this
specification uses the following terms:

{: vspace="0"}
Subject Identifier Object
: A JSON object containing a set of one or more claims about a subject that
  when taken together uniquely identify that subject. This set of claims
  SHOULD be declared as an acceptable way to identify subjects of SETs by
  one or more specifications that profile [SET](#SET).


Control Plane Resources {#resources}
=======================
Event receivers manage how they receive events and the subjects about which
they want to receive events by making HTTP requests to resources that
collectively make up the SET Control Plane.

Stream {#stream}
----------------
A stream represents a configured relationship between a single
event transmitter and a single event receiver, and describes which
events the transmitter may transmit to the receiver and how the
receiver may receive those events. A stream's event transmitter
transmits events over the stream to the stream's event receiver, making
the events available to the event receiver in an agreed upon way as
described by the stream's configuration.

Transmitters MAY support multiple streams for the same receiver.
Transmitters MAY also assign distinct URLs for different streams, and/or
provide a single URL for multiple streams, provided that the transmitter
has some mechanism through which they can identify the receiver and
stream, e.g. from authentication credentials. The definition of such
mechanisms is outside the scope of this specification.

Event streams have the following properties:

{: vspace="0"}
aud
: A string containing an audience claim as defined in [JSON Web Token
  (JWT)](#RFC7519) that identifies the event receiver for the stream.

events
: OPTIONAL. An array of URIs identifying the set of events which MAY be
delivered over the event stream. If omitted, transmitters SHOULD make this
set available to the receiver via some other means (e.g. publishing it in
online documentation). The value of this property is dictated by the
transmitter, and therefore it SHALL be omitted by the receiver when making
requests to modify the stream's configuration.

delivery_methods
: A JSON object containing a set of name/value pairs, where the name is a
URI identifying a SET delivery method as defined in [DELIVERY](#DELIVERY), and
the value is a JSON object whose name/value pairs contain the additional
configuration parameters for that SET delivery method. The value object may
be an empty JSON object (e.g. `{}`) if the SET delivery method does not have
any configuration parameters.

<<TODO: Update with actual SET delivery reference, or drop reference if we
merge docs.>>

### Reading a Stream's Configuration
An event receiver gets the current configuration of a stream by making
an HTTP GET request to the stream resource. On receiving a valid request
the event transmitter responds with a 200 OK response containing a {{!JSON}}
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
Content-Type: application/json; charset=UTF-8
Cache-Control: no-store
Pragma: no-cache

{
  "aud": "http://www.example.com",
  "delivery_methods": {
    "https://schemas.example.com/set/http-push": {
      "url": "https://receiver.example.com/events/risc"
    },
    "https://schemas.example.com/set/http-pull": {
      "url": "https://transmitter.example.com/events/risc"
    }
  },
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

Subjects {#subjects}
--------------------
Each stream has a set of subjects, entities or objects about which the
event transmitter may transmit events over the stream. Event receivers
can manipulate this set to manage which subjects they may receive events
about over the stream.

### Adding a Subject to a Stream
When a receiver wants to signal to a transmitter that they wish to receive
events about a particular subject over a stream, the receiver makes an HTTP
POST request to the `/add-subjects` endpoint under the stream's resource,
containing in the body a Subject Identifier Object identifying the subject
to be added. The transmitter MAY choose to ignore the request, for example
if the subject has previously indicated to the transmitter that they do not
want events to be transmitted to the receiver. On a successful response, the
transmitter responds with an empty 200 OK response.

<<TODO: MUST transmitters indicate that they are ignoring the request?>>

<<TODO: Errors>>

The following is a non-normative example request to add a subject to a
stream, where the subject is identified by OpenID Connect iss and sub
claims:

~~~
POST /streams/risc/add-subject HTTP/1.1
Host: transmitter.example.com
Authorization: Bearer eyJ0b2tlbiI6ImV4YW1wbGUifQo=

{
  "iss": "http://account.example.com",
  "sub": "1234567"
}
~~~

The following is a non-normative example response to a successful request:

~~~
HTTP/1.1 200 OK
Server: transmitter.example.com
Cache-Control: no-store
Pragma: no-cache

~~~

### Removing a Subject
When a receiver wants to signal to a transmitter that the receiver no
longer wishes to receive events about a subject over a stream, the
receiver makes an HTTP POST request to the `/remove-subject` endpoint under
the stream's resource, containing in the body a Subject Identifier Object
identifying the subject to be removed. On a successful response, the
transmitter responds with a 204 No Content response.

<<TODO: Errors>>

The following is a non-normative example request where the subject is
identified by an email claim:

~~~
POST /streams/risc/remove-subject HTTP/1.1
Host: transmitter.example.com
Authorization: Bearer eyJ0b2tlbiI6ImV4YW1wbGUifQo=

{
  "email": "example.user@example.com"
}
~~~

The following is a non-normative example response to a successful request:

~~~
HTTP/1.1 204 No Content
Server: transmitter.example.com
Cache-Control: no-store
Pragma: no-cache

~~~

Verification {#verify}
----------------------
In some cases, the frequency of event transmission on a stream will be
very low, making it difficult for an event receiver to tell the difference
between expected behavior and event transmission failure due to a
misconfigured stream. Event receivers can use a stream's `/verification`
resource to request that a verification event be transmitted over the
stream, allowing the receiver to confirm that the stream is configured
correctly upon successful receipt of the event.

{: vspace="0"}
state
: OPTIONAL. An arbitrary string that the transmitter MUST echo back to
  the receiver in the verification event's payload. Receivers MAY use
  the value of this parameter to correlate a verification event with a
  verification request.

### Triggering a Verification Event.
To request that a verification event be sent over a stream, the event
receiver makes an HTTP POST request to the stream's `/verification`
subresource, with a JSON object containing the parameters of the
verification request, if any. On a successful request, the event
transmitter responds with an empty 204 No Content response.

A successful response from a POST to the `/verification` subresource
does not indicate that the verification event was transmitted
successfully, only that the transmitter has transmitted the event or
will do so at some point in the future. Transmitters MAY transmit the
event via an asynchronous process, and SHOULD publish an SLA for
verification event transmission times. Receivers MUST NOT depend on the
verification event being transmitted synchonrously with their request.

<<TODO: Errors>>

The following is a non-normative example request to trigger a
verification event:

~~~
POST /streams/risc/verification HTTP/1.1
Host: transmitter.example.com
Authorization: Bearer eyJ0b2tlbiI6ImV4YW1wbGUifQo=
Content-Type: application/json; charset=UTF-8

{
  state: "VGhpcyBpcyBhbiBleGFtcGxlIHN0YXRlIHZhbHVlLgo="
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

--- back

