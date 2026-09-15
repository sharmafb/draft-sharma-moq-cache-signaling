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
QUIC Transport (MOQT). A subscriber can query the cache status of a finite
Track range or request the cache status of a FETCH. Responses report a hit,
miss, or partial hit and can identify locally cached ranges.


--- middle

# Introduction {#introduction}

Media over QUIC Transport (MOQT) {{MOQT}} permits Relays to cache Objects, but
does not expose local-cache coverage to subscribers. That information can aid
relay selection, rewind and representation choices, deadline-sensitive
retrieval, measurement, and debugging.

This document defines a cache query on TRACK_STATUS and cache reporting on
FETCH. Both return the same status and cached-range format and describe only
the responding endpoint. The signals are advisory and do not define cache
policy or change SUBSCRIBE or FETCH processing.

This document defines no new MOQT message types. It defines one Setup Option
and three Message Parameters.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the terms Object, Group, Track, Location, Publisher,
Subscriber, Relay, Message Parameter, and Setup Option as defined in {{MOQT}}.

This specification is based on version 18 of {{MOQT}}, identified by the
`moqt-18` protocol identifier. Its use with another version of MOQT is
undefined unless that version or a later revision of this document explicitly
declares compatibility.

Local Cache:
: Storage controlled by the responding endpoint that contains a complete
  Normal Object and its associated metadata. An Object is locally cached only
  if the endpoint can use it without initiating or waiting for an upstream
  MOQT operation.

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
| 0x01 | Cache Status queries |
| 0x02 | Fetch Cache Status reports |

An absent option is equivalent to a value of zero. Unknown capability bits
MUST be ignored.

A capability is negotiated when both endpoints include its bit. An endpoint
MUST NOT use the corresponding parameters otherwise.

The 0-RTT requirements of {{MOQT}} apply. In particular, a client MUST NOT use
these parameters in 0-RTT unless it has remembered that the server supports the
applicable capability.


# Cache Status {#cache-status}

CACHE_STATUS is a length-prefixed Message Parameter with Parameter Type TBD4.
It reports local-cache coverage using the following format. It MUST NOT appear
outside TRACK_STATUS_OK or FETCH_OK, or without the corresponding request.

~~~
CACHE_STATUS Value {
  Cache Status (vi64),
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

Cache Status has the following values:

| Value | Name | Meaning |
|---:|:-----|:--------|
| 0x00 | MISS | No Normal Object in scope was covered by the Local Cache. |
| 0x01 | PARTIAL | At least one Normal Object was covered, but the entire scope was not satisfied from local state. |
| 0x02 | HIT | The entire scope was satisfied from local state and included at least one Normal Object. |

A HIT MUST NOT include an Unknown range or require upstream activity. Known
non-existent Objects do not prevent a HIT when the Publisher can report them
authoritatively from local state. A FETCH response containing no Normal Objects
is a MISS.

For TRACK_STATUS_OK, the scope is the range in CACHE_STATUS_REQUEST and a
Cached Range identifies Objects present in the Local Cache when the request is
processed. For FETCH_OK, the scope is the completed FETCH response and a
Cached Range identifies Objects served from the Local Cache. An Object that
becomes available after FETCH processing begins is not considered cached for
that FETCH.

Range List Complete is 1 when every Cached Range in scope is present and 0
when any were omitted. Number of Cached Ranges MUST NOT exceed Maximum Cached
Ranges from the request. A Publisher MAY truncate the list because of that
limit or the maximum MOQT control-message size, but MUST then set Range List
Complete to 0. Cache Status always covers the full scope, even when the list
is truncated.

Cached Ranges MUST be ordered by increasing Group ID and Object ID, MUST NOT
overlap, and MUST combine adjacent Objects in the same Group. Last Object ID
MUST be at least First Object ID.

For MISS, Number of Cached Ranges MUST be zero and Range List Complete MUST be
one. If Maximum Cached Ranges is zero for a PARTIAL or HIT, both Number of
Cached Ranges and Range List Complete MUST be zero.

When Range List Complete is 1, a Normal Object in scope but outside the listed
ranges was not covered by the Local Cache. This extension does not identify
its source. When Range List Complete is 0, omission conveys no information
about an individual Object.

Any other Cache Status or Range List Complete value, or any malformed,
duplicated, overlapping, or out-of-scope range, is a PROTOCOL_VIOLATION.


# Cache Status Query {#cache-status-query}

CACHE_STATUS_REQUEST is a length-prefixed Message Parameter with Parameter
Type TBD2. It MAY appear exactly once in TRACK_STATUS, MUST NOT appear in any
other message, and has this value:

~~~
CACHE_STATUS_REQUEST Value {
  Start Location (Location),
  End Location (Location),
  Maximum Cached Ranges (vi64),
}
~~~

The Locations use the range semantics of a Standalone Fetch in {{MOQT}}.
Maximum Cached Ranges limits the number of ranges in CACHE_STATUS. When it is
zero, the Publisher returns no Cached Range entries but still returns the other
CACHE_STATUS fields.

The Publisher MUST apply the authorization checks used for a FETCH of the same
Track and range. It responds with REQUEST_ERROR and INVALID_RANGE when End
Location precedes Start Location, or UNAUTHORIZED when authorization fails.

If the request succeeds, the Publisher MUST include exactly one CACHE_STATUS
in TRACK_STATUS_OK, computed from a single cache snapshot. A Relay MUST NOT
contact an upstream endpoint solely to answer the query. An endpoint unable to
determine the status MUST reject the request rather than guess.

The result does not reserve Objects. Cache changes can make it stale before a
subsequent FETCH.


# Fetch Cache Status {#fetch-cache-status}

FETCH_CACHE_STATUS_REQUEST is a length-prefixed Message Parameter with
Parameter Type TBD3. It MAY appear exactly once in FETCH, MUST NOT appear in
any other message, and has this value:

~~~
FETCH_CACHE_STATUS_REQUEST Value {
  Maximum Cached Ranges (vi64),
}
~~~

Maximum Cached Ranges has the meaning defined in {{cache-status-query}}. A
Publisher that successfully processes the FETCH MUST include exactly one
CACHE_STATUS in FETCH_OK. It MUST NOT send FETCH_OK until the final status and
ranges are known, but MAY send Objects first as permitted by {{MOQT}}. A failed
FETCH has no CACHE_STATUS.

FETCH_CACHE_STATUS_REQUEST does not alter the requested range, Group Order,
FILL_TIMEOUT, Object payloads, or other FETCH behavior.


# Relay Processing {#relay-processing}

Message Parameters are hop-by-hop under {{MOQT}}. A Relay MUST NOT copy
CACHE_STATUS_REQUEST, FETCH_CACHE_STATUS_REQUEST, or CACHE_STATUS between
upstream and downstream messages. It MAY make an independent upstream request
when the capability is negotiated on that Session, but its downstream
CACHE_STATUS describes only its own Local Cache. The source of all other
Objects is outside the scope of this extension.


# Interaction with Other MOQT Mechanisms

FILL_TIMEOUT continues to control how long a Relay waits for Objects. A value
of zero can expose availability by transferring immediately available Objects;
CACHE_STATUS_REQUEST provides a snapshot without transferring payloads.

CACHE_DISTANCE {{CACHE_DISTANCE}} provides per-Object, multi-hop attribution.
CACHE_STATUS instead reports local coverage at the immediate peer and does not
identify non-cache sources. Cache changes can cause the two signals to differ.

This document does not define proactive cache advertisements or cache-aware
routing. CACHE_STATUS_REQUEST is intended for occasional, range-specific use.


# Operational Considerations

A CACHE_STATUS in TRACK_STATUS_OK is a snapshot, not a reservation or a
promise about a later FETCH. Applications SHOULD query the smallest useful
range. A Publisher MAY reject excessive requests with REQUEST_ERROR using
EXCESSIVE_LOAD.


# Security and Privacy Considerations {#security}

The security considerations of {{MOQT}} apply.

Cache signaling can reveal retained content and recent request patterns,
enabling cache probing or audience inference. A Publisher MUST apply the
authorization policy for the Track before returning CACHE_STATUS. Deployments
MAY truncate ranges, limit requests, or disable this extension across trust
boundaries.

Cache status is not authenticated end-to-end. Subscribers MUST treat it as
advisory and MUST NOT rely on it for authorization, content integrity, or
application correctness. Publishers SHOULD limit response size and rate-limit
cache queries.

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
| TBD2 | CACHE_STATUS_REQUEST | TRACK_STATUS | This document, {{cache-status-query}} |
| TBD3 | FETCH_CACHE_STATUS_REQUEST | FETCH | This document, {{fetch-cache-status}} |
| TBD4 | CACHE_STATUS | TRACK_STATUS_OK, FETCH_OK | This document, {{cache-status}} |

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
