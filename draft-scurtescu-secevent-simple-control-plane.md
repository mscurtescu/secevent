---
title: Management API for SET Event Streams
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
    # TODO: Update with real reference.
  DELIVERY:
    title: SET Token Delivery Using HTTP
    target: https://github.com/independentid/Identity-Events/blob/master/draft-hunt-secevent-delivery.txt
    # TODO: Update with real reference.

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
Event Stream
: An Event Stream is a configured relationship between a single Event
  Transmitter and a single Event Receiver, describing one or more methods by
  which the Event Transmitter may deliver SETs to the Event Receiver. Event
  Streams are unidirectional, with only one Event Transmitter and one Event
  Receiver. Event Transmitters MAY support multiple Event Streams for a
  single Event Receiver.

Event Stream Management Endpoint
: A URL hosted by the transmitter, which serves as the base for the stream
  management API for a stream. An Event Transmitter MAY use a single
  Management Endpoint for multiple streams, provided that the transmitter
  has some mechanism through which they can identify the applicable stream
  for any given request, e.g. from authentication credentials. The
  definition of such mechanisms is outside the scope of this specification.

Subject Identifier Object
: A JSON object containing a set of one or more claims about a subject that
  when taken together uniquely identify that subject. This set of claims
  SHOULD be declared as an acceptable way to identify subjects of SETs by
  one or more specifications that profile [SET](#SET).


Event Stream Management {#management}
=======================
Event Receivers manage how they receive events, and the subjects about which
they want to receive events over an Event Stream by making HTTP requests to
endpoints under the stream's Event Stream Management Endpoint. These
endpoints collectively make up the Event Stream Management API.

Stream Configuration {#stream}
--------------------
An Event Stream's configuration is represented as a JSON object with the
following properties:

{: vspace="0"}
aud
: A string containing an audience claim as defined in [JSON Web Token
  (JWT)](#RFC7519) that identifies the Event Receiver for the Event Stream.

events
: OPTIONAL. An array of URIs identifying the set of events which MAY be
delivered over the Event Stream. If omitted, Event Transmitters SHOULD make
this set available to the Event Receiver via some other means (e.g.
publishing it in online documentation).

delivery_methods
: A JSON object containing a set of name/value pairs, where the name is a
URI identifying a SET delivery method as defined in [DELIVERY](#DELIVERY), and the
value is a JSON object whose name/value pairs contain the additional
configuration parameters for that SET delivery method. The value object may
be an empty JSON object (e.g. `{}`) if the SET delivery method does not have
any configuration parameters.

### Reading a Stream's Configuration
An Event Receiver gets the current configuration of a stream by making an
HTTP GET request to the stream's Management Endpoint. On receiving a valid
request the Event Transmitter responds with a 200 OK response containing a
{{!JSON}} representation of the stream's configuration in the body.

The following is a non-normative example request to read an Event Stream's
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
An Event Receiver can indicate to an Event Transmitter whether or not the
receiver wants to receive events about a particular subject by "adding" or
"removing" that subject to the Event Stream, respectively.

### Adding a Subject to a Stream
To add a subject to an Event Stream, the Event Receiver makes an HTTP POST
request to the `/add-subjects` endpoint under the stream's Management
Endpoint, containing in the body a Subject Identifier Object identifying the
subject to be added. On a successful response, the Event Transmitter
responds with an empty 200 OK response.

The Event Transmitter MAY choose to silently ignore the request, for example
if the subject has previously indicated to the transmitter that they do not
want events to be transmitted to the Event Receiver. In this case, the
transmitter MUST return an empty 200 OK response, and MUST NOT indicate to
the receiver that the request was ignored.

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
To remove a subject from an Event Stream, the Event Receiver makes an HTTP
POST request to the `/remove-subject` endpoint under the stream's Management
Endpoint, containing in the body a Subject Identifier Object identifying the
subject to be removed. On a successful response, the Event Transmitter
responds with a 204 No Content response.

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
In some cases, the frequency of event transmission on an Event Stream will
be very low, making it difficult for an Event Receiver to tell the
difference between expected behavior and event transmission failure due to a
misconfigured stream. Event Receivers can request that a verification event
be transmitted over the Event Stream, allowing the receiver to confirm that
the stream is configured correctly upon successful receipt of the event.

Verification requests have the following properties:

{: vspace="0"}
state
: OPTIONAL. An arbitrary string that the Event Transmitter MUST echo back to
  the Event Receiver in the verification event's payload. Event Receivers
  MAY use the value of this parameter to correlate a verification event with
  a verification request.

### Triggering a Verification Event.
To request that a verification event be sent over an Event Stream, the Event
Receiver makes an HTTP POST request to the `/verification` endpoint under
the stream's Management Endpoint, with a JSON object containing the
parameters of the verification request, if any. On a successful request, the
event transmitter responds with an empty 204 No Content response.

A successful response from a POST to the `/verification` endpoint does not
indicate that the verification event was transmitted successfully, only that
the Event Transmitter has transmitted the event or will do so at some point
in the future. Event Transmitters MAY transmit the event via an asynchronous
process, and SHOULD publish an SLA for verification event transmission
times. Event Receivers MUST NOT depend on the verification event being
transmitted synchonrously with their request.

<<TODO: Errors>>

The following is a non-normative example request to trigger a verification
event:

~~~
POST /streams/risc/verification HTTP/1.1
Host: transmitter.example.com
Authorization: Bearer eyJ0b2tlbiI6ImV4YW1wbGUifQo=
Content-Type: application/json; charset=UTF-8

{
  state: "VGhpcyBpcyBhbiBleGFtcGxlIHN0YXRlIHZhbHVlLgo="
}
~~~

The following is a non-normative example response to a successful request:

~~~
HTTP/1.1 204 No Content
Server: transmitter.example.com
Cache-Control: no-store
Pragma: no-cache

~~~

--- back

