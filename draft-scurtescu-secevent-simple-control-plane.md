---
title: Management API for SET Event Streams
abbrev: set-control-plane
docname: draft-scurtescu-secevent-simple-control-plane-00
date: 2017-06-29
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
events, and how they wish to receive them. This specification defines an HTTP 
API for a basic control plane that event transmitters can implement and
event receivers may use to manage the flow of events from one to the
other.

--- middle

Introduction {#intro}
============
This specification defines an HTTP API to be implemented by Event Transmitters
and that can be used by Event Receivers to query the Event Stream status, to
add and remove subjects and to trigger verification.

~~~
+------------+                +------------+
|            | Stream Status  |            |
| Event      <----------------+ Event      |
| Stream     |                | Receiver   |
| Management | Add Subject    |            |
| API        <----------------+            |
|            |                |            |
|            | Remove Subject |            |
|            <----------------+            |
|            |                |            |
|            | Verification   |            |
|            <----------------+            |
|            |                |            |
+------------+                +------------+
~~~
{: #figintro title="Event Stream Management API"}

How events are delivered and the structure of events are not in scope for this
specification.

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

Event Stream Management {#management}
=======================
Event Receivers manage how they receive events, and the subjects about which
they want to receive events over an Event Stream by making HTTP requests to
endpoints in the Event Stream Management API.

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

delivery
: A JSON object containing a set of name/value pairs specifying configuration
parameters for the SET delivery method. The actual delivery method is
identified by the special key "delivery_method" with the value being a URI as
defined in [DELIVERY](#DELIVERY).

status
: A string indicating the current status of the event stream. It MUST have one 
of the following values:

  {: vspace="0"}
  enabled
  : The transmitter will transmit events over the stream, according to the
  stream's configured delivery method.

  paused
  : The transmitter will not transmit events over the stream. The transmitter
  will hold any events it would have transmitted while paused, and will
  transmit them when the stream's status becomes "enabled".

  disabled
  : The transmitter will not transmit events over the stream, and will not
  hold any events for later transmission.
  
### Checking a Stream's Status
An Event Receiver checks the current status of an event stream by making an
HTTP GET request to the stream's status endpoint. On receiving a valid request
the Event Transmitter responds with a 200 OK response containing a {{!JSON}}
object with a single attribute "status", whose string value is the value of
the stream's status.

The following is a non-normative example request to check an event stream's
status:

~~~
GET /set/stream/status HTTP/1.1
Host: transmitter.example.com
Authorization: Bearer eyJ0b2tlbiI6ImV4YW1wbGUifQo=
~~~
{: #figstatusreq title="Example: Check Stream Status Request"}

The following is a non-normative example response:

~~~
HTTP/1.1 200 OK
Content-Type: application/json; charset=UTF-8
Cache-Control: no-store
Pragma: no-cache

{
  "status": "enabled"
}
~~~
{: #figstatusresp title="Example: Check Stream Status Response"}

### Reading a Stream's Configuration
An Event Receiver gets the current configuration of a stream by making an
HTTP GET request to the configuration endpoint. On receiving a valid request
the Event Transmitter responds with a 200 OK response containing a {{!JSON}}
representation of the stream's configuration in the body.

The following is a non-normative example request to read an Event Stream's
configuration:

~~~
GET /set/stream HTTP/1.1
Host: transmitter.example.com
Authorization: Bearer eyJ0b2tlbiI6ImV4YW1wbGUifQo=
~~~
{: #figconfigreq title="Example: Read Stream Configuration Request"}

The following is a non-normative example response:

~~~
HTTP/1.1 200 OK
Content-Type: application/json; charset=UTF-8
Cache-Control: no-store
Pragma: no-cache

{
  "aud": "http://www.example.com",
  "delivery": {
    "delivery_method": "urn:example:secevent:delivery:http_post",
    "url": "https://receiver.example.com/events"
  },
  "events": [
    "urn:example:secevent:events:type_1",
    "urn:example:secevent:events:type_2",
    "urn:example:secevent:events:type_3"
  ]
}
~~~
{: #figconfigresp title="Example: Read Stream Configuration Response"}

Subjects {#subjects}
--------------------
An Event Receiver can indicate to an Event Transmitter whether or not the
receiver wants to receive events about a particular subject by "adding" or
"removing" that subject to the Event Stream, respectively.

### Adding a Subject to a Stream
To add a subject to an Event Stream, the Event Receiver makes an HTTP POST
request to the add subject endpoint, containing in the body a Subject
Identifier Object identifying the subject to be added. On a successful
response, the Event Transmitter responds with an empty 200 OK response.

The Event Transmitter MAY choose to silently ignore the request, for example
if the subject has previously indicated to the transmitter that they do not
want events to be transmitted to the Event Receiver. In this case, the
transmitter MUST return an empty 200 OK response, and MUST NOT indicate to
the receiver that the request was ignored.

Errors are signaled with HTTP staus codes as follows:

| Code | Description |
|------+-------------|
| 400  | if the request body cannot be parsed or if the request is otherwise invalid |
| 401  | if authorization failed or it is missing |
| 403  | if the Event Receiver is not allowed to add this particular subject |
| 404  | if the subject is not recognized by the Event Transmitter, the Event Transmitter may chose to stay silent in this case and responde with 200 |
| 429  | if the Event Receiver is sending too many requests in a gvien amount of time |
{: #tabadderr title="Add Subject Errors"}

The following is a non-normative example request to add a subject to a
stream, where the subject is identified by an OpenID Connect email claim:

~~~
POST /set/subjects:add HTTP/1.1
Host: transmitter.example.com
Authorization: Bearer eyJ0b2tlbiI6ImV4YW1wbGUifQo=

{
  "email": "example.user@example.com"
}
~~~
{: #figaddreq title="Example: Add Subject Request"}

The following is a non-normative example response to a successful request:

~~~
HTTP/1.1 200 OK
Server: transmitter.example.com
Cache-Control: no-store
Pragma: no-cache
~~~
{: #figaddresp title="Example: Add Subject Response"}

### Removing a Subject
To remove a subject from an Event Stream, the Event Receiver makes an HTTP
POST request to the Remove Subject endpoint, containing in the body a Subject
Identifier Object identifying the subject to be removed. On a successful
response, the Event Transmitter responds with a 204 No Content response.

Errors are signaled with HTTP staus codes as follows:

| Code | Description |
|------+-------------|
| 400  | if the request body cannot be parsed or if the request is otherwise invalid |
| 401  | if authorization failed or it is missing |
| 403  | if the Event Receiver is not allowed to remove this particular subject |
| 404  | if the subject is not recognized by the Event Transmitter, the Event Transmitter may chose to stay silent in this case and responde with 204 |
| 429  | if the Event Receiver is sending too many requests in a gvien amount of time |
{: #tabremoveerr title="Remove Subject Errors"}

The following is a non-normative example request where the subject is
identified by a phone_number claim:

~~~
POST /set/subjects:remove HTTP/1.1
Host: transmitter.example.com
Authorization: Bearer eyJ0b2tlbiI6ImV4YW1wbGUifQo=

{
  "phone_number": "+1 206 555 1234"
}
~~~
{: #figremovereq title="Example: Remove Subject Request"}

The following is a non-normative example response to a successful request:

~~~
HTTP/1.1 204 No Content
Server: transmitter.example.com
Cache-Control: no-store
Pragma: no-cache
~~~
{: #figremoveresp title="Example: Remove Subject Response"}

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
Receiver makes an HTTP POST request to the verification endpoint, with a JSON
object containing the parameters of the verification request, if any. On a
successful request, the event transmitter responds with an empty 204 No Content
response.

A successful response from a POST to the verification endpoint does not
indicate that the verification event was transmitted successfully, only that
the Event Transmitter has transmitted the event or will do so at some point
in the future. Event Transmitters MAY transmit the event via an asynchronous
process, and SHOULD publish an SLA for verification event transmission
times. Event Receivers MUST NOT depend on the verification event being
transmitted synchronously with their request.

Errors are signaled with HTTP staus codes as follows:

| Code | Description |
|------+-------------|
| 400  | if the request body cannot be parsed or if the request is otherwise invalid |
| 401  | if authorization failed or it is missing |
| 429  | if the Event Receiver is sending too many requests in a gvien amount of time |
{: #taberifyerr title="Verification Errors"}

The following is a non-normative example request to trigger a verification
event:

~~~
POST /set/verify HTTP/1.1
Host: transmitter.example.com
Authorization: Bearer eyJ0b2tlbiI6ImV4YW1wbGUifQo=
Content-Type: application/json; charset=UTF-8

{
  "state": "VGhpcyBpcyBhbiBleGFtcGxlIHN0YXRlIHZhbHVlLgo="
}
~~~
{: #figverifyreq title="Example: Trigger Verification Request"}

The following is a non-normative example response to a successful request:

~~~
HTTP/1.1 204 No Content
Server: transmitter.example.com
Cache-Control: no-store
Pragma: no-cache
~~~
{: #figverifyresp title="Example: Trigger Verification Response"}

And the following is a non-normative example of a verification event sent to
the Event Receiver as a result of the above request:

~~~
{
  "jti": "123456",
  "iss": "https://transmitter.example.com",
  "aud": "receiver.example.com",
  "iat": "1493856000",
  "events": [
    "urn:ietf:params:secevent:event-type:core:verify" : {
      "state": "VGhpcyBpcyBhbiBleGFtcGxlIHN0YXRlIHZhbHVlLgo=",
    },
  ],
}
~~~
{: #figverifyset title="Example: Verification SET"}

Security Considerations {#SecCon}
=======================

Subject Probing
---------------
It may be possible for an Event Transmitter to leak information about subjects
through their responses to add subject requests. A 404 response may indicate
to the Event Receiver that the subject does not exist, which may inadvertantly
reveal information about the subject (e.g. that a particular individual does
or does not use the Event Transmitter's service).

Event Transmitters SHOULD carefully evaluate the conditions under which they
will return error responses to add subject requests. Event Transmitters MAY
return a 204 response even if they will not actually send any events related
to the subject, and Event Receivers MUST NOT assume that a 204 response means
that they will receive events related to the subject.

Information Harvesting
----------------------
SETs may contain personally identifiable information (PII) or other non-public
information about the event transmitter, the subject (of an event in the SET),
or the relationship between the two. It is important for Event Transmitters to
understand what information they are revealing to Event Receivers when
transmitting events to them, lest the event stream become a vector for 
unauthorized access to private information.

Event Transmitters SHOULD interpret add subject requests as statements of
interest in a subject by an Event Receiver, and ARE NOT obligated to transmit
events related to every subject an Event Receiver adds to the stream. Event
Transmitters MAY choose to transmit some, all, or no events related to any
given subject and SHOULD validate that they are permitted to share the
information contained within an event with the Event Receiver before
transmitting the event. The mechanisms by which such validation is performed
are outside the scope of this specification.

Malicous Subject Removal
------------------------
A malicious party may find it advantageous to remove a particular subject from
a stream, in order to reduce the Event Receiver's ability to detect
malicious activity related to the subject, inconvenience the subject, or for
other reasons. Consequently it may be in the best interests of the subject for
the Event Transmitter to continue to send events related to the subject for
some time after the subject has been removed from a stream.

Event Transmitters MAY continue sending events related to a subject for some
amount of time after that subject has been removed from the stream. Event
Receivers MUST tolerate receiving events for subjects that have been removed
from the stream, and MUST NOT report these events as errors to the Event
Transmitter.

--- back

