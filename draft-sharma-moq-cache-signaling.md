---
title: "Cache Signaling for Media over QUIC Transport"
abbrev: "moq-cache-signaling"
category: std

docname: draft-sharma-moq-cache-signaling-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "Media Over QUIC"
keyword:
 - media over quic
 - cache
 - relay

venue:
  group: "Media Over QUIC"
  type: "Working Group"
  mail: "moq@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/moq/"
  github: "sharmafb/draft-sharma-moq-cache-signaling"
  latest: "https://sharmafb.github.io/draft-sharma-moq-cache-signaling/draft-sharma-moq-cache-signaling.html"

smart_quotes: no

author:
 -
    ins: A. Sharma
    fullname: Aman Sharma
    organization: Meta
    email: amsharma@meta.com

normative:
  MOQT: I-D.ietf-moq-transport

informative:
  CACHE_DISTANCE:
    title: "Cache Distance Property for MOQT"
    target: "https://afrind.github.io/draft-frindell-moq-cache-distance/draft-frindell-moq-cache-distance.html"
    author:
      -
        ins: A. Frindell
        name: Alan Frindell
    date: false

--- abstract

This document defines optional hop-by-hop cache signaling for Media over
QUIC Transport (MOQT). It allows an endpoint to query whether a finite
range of a Track is available in the local cache of its peer. It also
allows a subscriber to learn whether a FETCH response was a cache hit or
cache miss at the responding endpoint.

Cache signaling is advisory, represents only the responding endpoint,
and does not reserve cached Objects or change the processing of a
SUBSCRIBE or FETCH.


--- middle

# Introduction {#introduction}

Media over QUIC Transport (MOQT) {{MOQT}} permits Relays to cache Objects and
use those Objects to satisfy downstream requests. However, MOQT does not
provide a subscriber with a way to determine whether a particular range is
present in a Relay's local cache before requesting it. It also does not report
whether a successful FETCH was served locally or required upstream retrieval.

This information can be useful for:

* selecting between otherwise equivalent Relays;
* choosing a starting point for delayed-live or rewind playback;
* selecting an initially cached media representation;
* deciding whether to attempt retrieval of content subject to a short
  deadline;
* measuring cache effectiveness and diagnosing latency; and
* estimating the load imposed on upstream publishers.

This document defines two related mechanisms:

1. A Cache Availability query carried in TRACK_STATUS and TRACK_STATUS_OK.
2. A Fetch Cache Status report requested in FETCH and returned in FETCH_OK.

Both mechanisms describe only the endpoint that directly answers the request.
They are snapshots rather than promises. This document does not define cache
admission, eviction, replacement, prefetching, or routing policy.

This document defines no new MOQT message types. It defines one Setup Option
and four Message Parameters.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the terms Object, Group, Track, Location, Publisher,
Subscriber, Relay, Original Publisher, Message Parameter, and Setup Option as
defined in {{MOQT}}.

This specification is based on version 18 of {{MOQT}}, identified by the
`moqt-18` protocol identifier. Its use with another version of MOQT is
undefined unless that version or a later revision of this document explicitly
declares compatibility.

Local Cache:
: Storage controlled by the responding endpoint that contains a complete
  Normal Object and its associated metadata. An Object is locally cached only
  if the endpoint can use it without initiating or waiting for an upstream
  MOQT operation.

Cache Snapshot:
: The state of the Local Cache when the responding endpoint processes a Cache
  Availability request.

Cached Range:
: A contiguous sequence of locally cached Normal Objects having the same Group
  ID and consecutive Object IDs.

An Object whose MAX_CACHE_DURATION has elapsed, as defined by {{MOQT}}, is not
considered locally cached. Object Status records are not Normal Objects and are
not included in Cached Ranges.


# Extension Negotiation {#negotiation}

Because {{MOQT}} requires receipt of an unknown Message Parameter to terminate
the Session, use of this extension is negotiated with the CACHE_SIGNALING Setup
Option.

The CACHE_SIGNALING Setup Option has an even-numbered Option Type of TBD1 and a
variable-length integer value containing a capability bit mask.

| Bit | Capability |
|---:|:-----------|
| 0x01 | Cache Availability queries |
| 0x02 | Fetch Cache Status reports |

An absent option is equivalent to a value of zero. Unknown capability bits
MUST be ignored.

A capability is negotiated when both endpoints include its bit in their
CACHE_SIGNALING Setup Option. An endpoint MUST NOT send a Message Parameter
defined by this document unless its corresponding capability was negotiated.

The 0-RTT requirements of {{MOQT}} apply. In particular, a client MUST NOT use
these parameters in 0-RTT unless it has remembered that the server supports the
applicable capability.


# Cache Availability {#cache-availability}

Cache Availability provides an on-demand snapshot of the Objects in a finite
Track range that are present in the responding endpoint's Local Cache.

The query is hop-by-hop. A Relay MUST NOT forward the query or initiate an
upstream SUBSCRIBE, FETCH, or TRACK_STATUS solely to answer it.

## CACHE_AVAILABILITY_REQUEST Parameter {#availability-request}

CACHE_AVAILABILITY_REQUEST is a length-prefixed Message Parameter with
Parameter Type TBD2. It MAY appear exactly once in TRACK_STATUS and MUST NOT
appear in any other message.

Its value is:

~~~
CACHE_AVAILABILITY_REQUEST Value {
  Start Location (Location),
  End Location (Location),
  Maximum Ranges (vi64),
}
~~~

Start Location and End Location identify the range being queried. They use the
same range semantics as the corresponding fields of a Standalone Fetch in
{{MOQT}}.

Maximum Ranges is the largest number of Cached Range entries the requester is
willing to receive. A value of zero requests only the aggregate Availability
value.

If End Location precedes Start Location, the Publisher MUST respond with
REQUEST_ERROR using error code INVALID_RANGE.

A Publisher MUST apply the same authorization checks that it would apply to a
FETCH for the requested Track and range. An unauthorized request is rejected
with REQUEST_ERROR using error code UNAUTHORIZED.

## CACHE_AVAILABILITY Parameter {#availability-response}

A Publisher that accepts a TRACK_STATUS containing
CACHE_AVAILABILITY_REQUEST MUST include exactly one CACHE_AVAILABILITY
parameter in TRACK_STATUS_OK.

CACHE_AVAILABILITY is a length-prefixed Message Parameter with Parameter Type
TBD3. Its value is:

~~~
CACHE_AVAILABILITY Value {
  Availability (vi64),
  Range List Complete (8),
  Number of Cached Ranges (vi64),
  Cached Range (..) ...,
}

Cached Range {
  Group ID (vi64),
  First Object ID (vi64),
  Last Object ID (vi64),
}
~~~

Availability has the following values:

| Value | Name | Meaning |
|---:|:-----|:--------|
| 0x00 | UNKNOWN | The Publisher cannot or will not determine its local cache state. |
| 0x01 | NONE | No Normal Object in the requested range is locally cached. |
| 0x02 | PARTIAL | At least one Normal Object is locally cached, but the complete requested range cannot be served using local state. |
| 0x03 | FULL | The requested range can be served entirely using local state, without upstream activity or an Unknown range in the corresponding FETCH response. |

Known non-existent Objects do not prevent a range from being FULL, provided
the Publisher has sufficient local state to report those gaps authoritatively.

Range List Complete is 1 when the response includes every Cached Range in the
requested range and 0 when one or more ranges were omitted. Any other value is
a PROTOCOL_VIOLATION.

Number of Cached Ranges MUST NOT exceed Maximum Ranges from the request. A
Publisher MAY return fewer entries because of implementation limits or the
maximum MOQT control-message size. In that case, it MUST set Range List
Complete to 0.

Each Cached Range describes consecutive Normal Objects in one Group. Last
Object ID MUST be greater than or equal to First Object ID. Entries MUST be
ordered by increasing Group ID and Object ID, MUST NOT overlap, and MUST be
combined when adjacent entries from the same Group can be represented as one
range.

For UNKNOWN, Number of Cached Ranges MUST be zero and Range List Complete MUST
be zero. For NONE, Number of Cached Ranges MUST be zero and Range List Complete
MUST be one.

The Publisher computes Availability and Cached Ranges from a single Cache
Snapshot. Receipt of the response does not reserve any Object. An Object
reported as cached can be evicted before a subsequent FETCH, and an Object
reported as unavailable can enter the cache immediately after the response.

A recipient MUST treat malformed, overlapping, or out-of-range Cached Ranges
as a PROTOCOL_VIOLATION.


# Fetch Cache Status {#fetch-cache-status}

Fetch Cache Status reports whether a completed FETCH response was a cache hit
or cache miss at the responding Publisher.

It does not change the requested range, Group Order, FILL_TIMEOUT, Object
payload, or any other FETCH behavior.

## FETCH_CACHE_STATUS_REQUEST Parameter {#fetch-status-request}

FETCH_CACHE_STATUS_REQUEST is a length-prefixed Message Parameter with
Parameter Type TBD4. It MAY appear exactly once in FETCH and MUST NOT appear in
any other message.

The value of FETCH_CACHE_STATUS_REQUEST MUST be empty. The presence of the
parameter requests a fetch-level cache status in FETCH_OK.

## FETCH_CACHE_STATUS Parameter {#fetch-status-response}

A Publisher that successfully processes a FETCH containing
FETCH_CACHE_STATUS_REQUEST MUST include exactly one FETCH_CACHE_STATUS
parameter in FETCH_OK.

FETCH_CACHE_STATUS is a length-prefixed Message Parameter with Parameter Type
TBD5. Its value is:

~~~
FETCH_CACHE_STATUS Value {
  Cache Status (8),
}
~~~

Cache Status has the following values:

| Value | Name |
|---:|:-----|
| 0x00 | MISS |
| 0x01 | HIT |

HIT means that the FETCH was satisfied entirely using the responding
Publisher's local state. At least one Normal Object MUST have been serialized
in the response, every Normal Object serialized MUST have been complete in the
Local Cache when processing of the FETCH began, the response MUST NOT contain
an Unknown range, and the Publisher MUST NOT have initiated or waited for an
upstream MOQT operation to determine any part of the response. Authoritatively
known non-existent Objects do not prevent a response from being a HIT.

MISS means that the response does not meet every requirement for HIT. In
particular, a mixed response containing both locally cached and upstream
Objects is a MISS, as is a response containing no Normal Objects. Cache Status
does not identify which Objects caused the miss or where they were obtained.

A recipient MUST treat any other Cache Status value as a
PROTOCOL_VIOLATION.

Because FETCH_CACHE_STATUS describes the completed response, the Publisher
MUST NOT send FETCH_OK until it can determine the final Cache Status. This can
delay FETCH_OK. As permitted by {{MOQT}}, the Publisher MAY begin transmitting
Objects on the FETCH data stream before sending FETCH_OK.

If the FETCH is rejected or ultimately fails with REQUEST_ERROR, no
FETCH_CACHE_STATUS is returned. Existing MOQT errors and stream-reset codes
communicate that failure.


# Relay Processing {#relay-processing}

Message Parameters are hop-by-hop under {{MOQT}}. A Relay MUST NOT copy
CACHE_AVAILABILITY_REQUEST, CACHE_AVAILABILITY, FETCH_CACHE_STATUS_REQUEST, or
FETCH_CACHE_STATUS between downstream and upstream requests.

A Relay MAY independently request cache signaling from its upstream peer when
the extension is negotiated on that Session. Such an upstream response does
not determine the Relay's downstream response:

* Cache Availability always describes the Relay's own Local Cache.
* Fetch Cache Status describes whether that Relay's downstream FETCH response
  was a complete local cache hit. Content obtained from any upstream peer makes
  the response a MISS, regardless of whether that peer served it from its own
  cache.

Cache information is advisory. A subscriber MUST NOT use it to infer that an
Object exists, to override an authoritative indication that an Object does not
exist, or as a substitute for successful receipt of an Object.


# Interaction with Other MOQT Mechanisms

## FILL_TIMEOUT

A FETCH with FILL_TIMEOUT equal to zero requests only Objects that are
immediately available and causes unavailable Objects to be represented as
Unknown ranges, as specified by {{MOQT}}.

Such a FETCH can reveal some cache information, but it also transfers the
available Objects. Cache Availability allows a subscriber to ask for a cache
snapshot without fetching Object payloads.

FILL_TIMEOUT continues to control how long a Relay waits for unavailable
Objects. Fetch Cache Status only reports whether the resulting response was a
cache hit or miss.

## Cache Distance

CACHE_DISTANCE {{CACHE_DISTANCE}} is an Object Property that reports, for each
portion of a FETCH response, the number of hops to the cache that supplied the
Objects.

CACHE_DISTANCE and Fetch Cache Status provide related but distinct
information. CACHE_DISTANCE supplies fine-grained, composable, multi-hop
attribution on the FETCH data stream. Fetch Cache Status provides only an
explicitly requested, fetch-level hit or miss result for the immediate peer.

If both extensions are used, an inconsistency between their values is not
itself a MOQT protocol violation. Cache state can change while a request is
processed, and both signals are supplied by peers rather than authenticated by
the Original Publisher.

## Namespace Routing

This document does not define proactive cache advertisements on
PUBLISH_NAMESPACE or NAMESPACE and is not a routing protocol.

Cache Availability is intended for occasional, range-specific decisions.
Endpoints SHOULD NOT poll it continuously. A deployment requiring continuously
updated cache-aware route selection should use a namespace-scoped routing
extension.


# Operational Considerations

Cache Availability represents a point-in-time observation. Network delay and
normal eviction can make the result stale before the subscriber acts on it.
Implementations MUST NOT interpret FULL as a reservation or guarantee that a
later FETCH will be a cache hit.

A Publisher MAY return UNKNOWN, omit detailed ranges, or limit query frequency
to avoid expensive cache scans. It MAY reject excessive requests with
REQUEST_ERROR using error code EXCESSIVE_LOAD.

Applications ought to query the smallest useful range. Large ranges can
produce expensive cache lookups and fragmented range lists.


# Security and Privacy Considerations {#security}

The security considerations of {{MOQT}} apply.

Cache signaling can reveal whether other users have recently requested
particular content, the contents retained at a Relay, and aspects of a
deployment's topology or traffic distribution. This information can be used
for cache probing, audience inference, or cross-user correlation.

A Publisher MUST apply the authorization policy for the requested Track before
returning cache information. Authorization to learn that a Track exists does
not necessarily imply authorization to inspect detailed cache state.
Deployments MAY return UNKNOWN, omit ranges, coarsen results, or disable this
extension across trust boundaries.

Cache responses are not authenticated end-to-end. A malicious or faulty Relay
can report arbitrary values. Subscribers MUST treat the information as
advisory and MUST NOT rely on it for authorization, content integrity, or
application correctness.

Cache queries can consume CPU and storage-index resources. Publishers SHOULD
limit the number and size of returned ranges and SHOULD rate-limit repeated
queries. Implementations MAY reject requests with EXCESSIVE_LOAD.

The parameters defined here do not modify Object payloads and are safe to
replay in the same sense as their enclosing TRACK_STATUS or FETCH requests.
Replayed requests can nevertheless increase load or expose additional cache
observations, so the 0-RTT precautions in {{MOQT}} continue to apply.


# IANA Considerations {#iana}

This document requests registration of the following Setup Option in the
"MOQT Setup Options" registry:

| Type | Name | Specification |
|---:|:-----|:--------------|
| TBD1 (even) | CACHE_SIGNALING | This document, {{negotiation}} |

An even-numbered code point is requested because the option value is a
variable-length integer.

This document requests registration of the following Message Parameters in the
"MOQT Message Parameters" registry:

| Type | Name | Permitted Messages | Specification |
|---:|:-----|:-------------------|:--------------|
| TBD2 | CACHE_AVAILABILITY_REQUEST | TRACK_STATUS | This document, {{availability-request}} |
| TBD3 | CACHE_AVAILABILITY | TRACK_STATUS_OK | This document, {{availability-response}} |
| TBD4 | FETCH_CACHE_STATUS_REQUEST | FETCH | This document, {{fetch-status-request}} |
| TBD5 | FETCH_CACHE_STATUS | FETCH_OK | This document, {{fetch-status-response}} |

The code points are placeholders. Provisional assignments should be requested
before interoperable implementations use them.


--- back

# Acknowledgments
{:numbered="false"}

Thanks to Alan Frindell, Luke Curley, and Steven Riedl for discussion that
motivated and refined this proposal.

# Change Log
{:numbered="false"}

## draft-sharma-moq-cache-signaling-00
{:numbered="false"}

* Initial version.

# Use of Generative AI
{:numbered="false"}

OpenAI Codex was used to assist with drafting and editing this document. All
generated text was reviewed and approved by the author.
