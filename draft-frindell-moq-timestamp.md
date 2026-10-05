---
title: "Timestamp Properties for MOQT"
abbrev: "moq-timestamp"
category: std

docname: draft-frindell-moq-timestamp-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "Media Over QUIC"
keyword:
 - media over quic
 - timestamp
venue:
  group: "Media Over QUIC"
  type: "Working Group"
  mail: "moq@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/moq/"
  github: "afrind/draft-frindell-moq-timestamp"
  latest: "https://afrind.github.io/draft-frindell-moq-timestamp/draft-frindell-moq-timestamp.html"

author:
  -
    ins: A. Frindell
    name: Alan Frindell
    organization: Meta
    email: afrind@meta.com
  -
    ins: I. Swett
    name: Ian Swett
    organization: Google
    email: ianswett@google.com

normative:
  MOQT: I-D.ietf-moq-transport

informative:
  LOC: I-D.ietf-moq-loc
  MSF: I-D.ietf-moq-msf
  TIMESTAMP-LCURLEY: I-D.lcurley-moq-timestamp

--- abstract

This document defines a set of MOQT Properties for carrying per-Object
timestamps efficiently. The encoded timestamp is intended for use in MOQT,
but can be referenced for application specific purposes.

--- middle

# Introduction

Media over QUIC Transport (MOQT) {{MOQT}} delivers Tracks that contain a
sequence of Objects. Though the transport layer does not need to know
media-oriented or application level timestamps, timing information can
help it make optimal scheduling decisions.  Additionally, they provide
visibility into latency and offer a Property applications can extend.

This document defines how a MOQT timestamp is encoded.
The design has three features:

* **Initial time**: A Track declares its start time once, so Objects can delta
  encode their timestamps from the Initial time.

* **Default Inter-Group/Object timing**: A Track can define a mapping from Group ID
  and Object ID to a timestamp, conveying timing with no per-Object bytes at all.

* **Compact encoding**: Per-Object timestamps are integers, expressing
  either a delta value from the initial time or a correction to the default value.

## Relationship to Other Specifications {#related}

Several specifications already carry timing for MOQT Objects.  The Low Overhead
Media Container {{LOC}} defines Timestamp and Timescale Properties for media
carried in LOC.  {{TIMESTAMP-LCURLEY}} specifies transport-level use of those
same LOC Properties so that Relays can make age-based decisions.  The MOQT
Streaming Format {{MSF}} relates media time, wall-clock time, and Location
through catalog fields and timeline tracks, including a template for regular
cadences, and numbers Groups by capture time in its log and metrics tracks.

This document aims to provide a single, general representation that these and
other specifications can reference.

# Conventions and Definitions {#conventions-and-definitions}

{::boilerplate bcp14-tagged}

This document uses the terms Track, Object, Group, and Subgroup as defined in
{{MOQT}}.  A "tick" is one unit of the Track's Timescale (see {{timescale}}).

All Property values in this document are encoded as variable-length integers
({{MOQT}}) unless otherwise noted.

## Signed Integer Zig-Zag Encoding {#zig-zag}

Signed values, such as the timestamp correction ({{object-timestamp}}), are
carried in a variable-length integer using a zig-zag mapping that keeps
small-magnitude values short: non-negative and negative values are interleaved
so that the encoded value grows with the magnitude, in the order 0, -1, 1, -2,
2, ...

To encode a signed value v as the variable-length integer u, and to decode it
back (both using an arithmetic, sign-extending right shift):

~~~
  u = (v << 1) ^ (v >> (WIDTH - 1))     ; encode
  v = (u >> 1) ^ -(u & 1)               ; decode
~~~

WIDTH is the bit width of the two's-complement representation of v (for example,
64).  Values outside the range -2^63 to 2^63-1 cannot be represented.

# Property Handling and Encoding {#property-handling}

The Properties defined in this document are serialized as Key-Value-Pairs
{{MOQT}}.

Each Property defined here MUST appear at most once on a given Track or Object,
counting both the mutable list and Immutable Properties ({{MOQT}}), and MUST
appear only in its defined scope: OBJECT_TIMESTAMP MUST NOT appear as a Track
Property, and the Track Properties (TIMESCALE, CLOCK_ID, TIMESTAMP_ORIGIN,
TIMESTAMP_MAPPING) MUST NOT appear as Object Properties.  A subscriber that
receives a Track or Object that violates these rules treats the track as
malformed, as specified in {{MOQT}}.

These Properties are set by the Original Publisher.  Relays MUST NOT add,
modify, or remove them.  A publisher MAY carry them in Immutable Properties
({{MOQT}}), for example to enable end-to-end authentication of timing.

Because the Properties defined here are interdependent, an endpoint that
interprets any of them MUST implement all of them.

# Track Properties {#track-properties}

A Track that uses the timestamps defined in this document declares a Timescale
({{timescale}}) and, optionally, a Clock ID ({{clock-id}}) and Timestamp Origin
({{timestamp-origin}}) that place its timeline on a clock.  All Object
timestamps in the Track are interpreted against this clock.

## Timescale {#timescale}

TIMESCALE is a Track Property giving the number of ticks per second used by all
timestamps in the Track.  Common values are 1000 for millisecond resolution and
1000000 for microsecond resolution, but any positive value MAY be used (for
example, a media Track might use its codec sample rate).

There is no default Timescale, to avoid silent unit errors such as confusing
milliseconds with microseconds.  A subscriber that receives a Track
with other Properties defined in this document but no TIMESCALE, or a TIMESCALE
value of 0, treats the Track as malformed.

## Clock ID {#clock-id}

CLOCK_ID is a Track Property identifying the clock on which the Track's timeline
is placed:

* A CLOCK_ID of 0 identifies wall-clock time, measured as POSIX time (seconds
  since 1970-01-01T00:00:00Z, excluding leap seconds).

* A non-zero CLOCK_ID identifies a clock with no defined relationship to
  wall-clock time.  Tracks that carry the same non-zero CLOCK_ID share that
  clock, so their timestamps can be compared -- for example, the audio and video
  Tracks of an on-demand asset.

* If CLOCK_ID is absent, the Track's timeline is not placed on any clock: its
  timestamps are meaningful only relative to one another within the Track.

A non-zero CLOCK_ID identifies the same clock wherever it appears, so values
chosen independently by different publishers can collide.  A publisher SHOULD
choose values from a large space (at least 62 bits) in a way that makes
accidental collisions negligible without coordination.  A value can be random,
or derived deterministically -- for example, by hashing a stable identifier for
the content -- so that separate encoders, or a publisher that restarts, use the
same value for the same clock.

## Timestamp Origin {#timestamp-origin}

TIMESTAMP_ORIGIN is a Track Property giving the position, in ticks on the
Track's clock ({{clock-id}}), that corresponds to a timestamp of 0.  An Object's
time on that clock (in TIMESCALE ticks) is:

~~~
  clock_time = timestamp_origin + object_timestamp
~~~

Because each Track's origin and timestamps are counted in its own ticks, Tracks
with different Timescales are compared by converting clock_time to seconds.

If TIMESTAMP_ORIGIN is absent, the default value is 0.  A Track that carries
TIMESTAMP_ORIGIN without CLOCK_ID is malformed.

## Timestamp Mapping {#timestamp-mapping}

TIMESTAMP_MAPPING is a Track Property that defines how to compute an Object's
timestamp from its Group ID and Object ID, with no per-Object Property.
Drift can be expressed with a property on any Object ({{object-timestamp}}).

The property value is four variable-length integers: a Base Group ID,
a Base Timestamp, a Group Multiplier, and an Object Multiplier.
A value that does not parse as exactly four variable-length integers is
malformed.  An Object's mapped timestamp is a linear function of its Group ID
and Object ID:

~~~
  mapped_timestamp = base_timestamp
                   + (group_id - base_group) * group_multiplier
                   + object_id * object_multiplier
~~~

The computation uses signed arithmetic, so it applies to every Group, including
Groups before the Base Group.  A negative mapped_timestamp is valid only if a
correction brings the Object's timestamp to a non-negative value.

The publisher chooses the values from the meaning it gives its Group and Object
identifiers:

* The **Group Multiplier** converts a Group ID into the Group's start time.  Set
  it to 1 when Group IDs are themselves timestamps in ticks, so each Group is
  placed directly by its ID; or to the number of ticks per Group when Group IDs
  are sequential indices and Groups have a fixed duration.

* The **Object Multiplier** converts an Object ID into an offset within its
  Group, giving the per-Object cadence, or 0 when every Object in a Group shares
  the Group's time.

* The **Base Group** and **Base Timestamp** anchor the mapping, so that a
  publisher whose Group IDs do not start at 0 -- for example, one that begins
  numbering at a wall-clock value and increments by one -- can still use a fixed
  Group Multiplier.  Both are 0 when Group 0 starts at timestamp 0.

A publisher can thus rely on the mapping for the regular majority of Objects and
spend per-Object bytes only where an Object's timestamp differs from the
schedule.

# Object Timestamp {#object-timestamp}

OBJECT_TIMESTAMP is an Object Property that conveys the Object's timestamp, in
ticks of the Track's Timescale.  How its value is interpreted depends on whether
the Track has a Timestamp Mapping ({{timestamp-mapping}}):

* On a Track without a Timestamp Mapping, the value is the Object's timestamp,
  encoded as an unsigned variable-length integer.  An Object that does not carry
  OBJECT_TIMESTAMP has no timestamp.

* On a Track with a Timestamp Mapping, the value is a signed correction to the
  Object's mapped timestamp, encoded using the zig-zag mapping in
  {{zig-zag}}.  An Object that does not carry OBJECT_TIMESTAMP takes its mapped
  timestamp:

  ~~~
    object_timestamp = mapped_timestamp + correction
  ~~~

Because a Track's Properties are known before any of its Objects, a receiver
always knows which rule applies.

The timestamp of an Object is computed only from Track Properties and
properties on the Object itself, and not any other Object.  This allows for
correct computation even when Objects are filtered or arrive out of order.

If a subscriber computes an Object's timestamp that is less than 0 or greater
than 2^64-1, it treats the Track as malformed.

# Locating Objects by Time {#time-to-location}

When a Track has a Timestamp Mapping with a non-zero Group Multiplier, a
receiver can estimate the Location of the Object with a given timestamp t
without receiving any Object:

~~~
  group_id  = base_group
            + floor((t - base_timestamp) / group_multiplier)
  object_id = floor((t - base_timestamp
                     - (group_id - base_group) * group_multiplier)
                    / object_multiplier)
~~~

If the Object Multiplier is 0, only the Group is estimated.  For a time c on the
Track's clock ({{clock-id}}), t is c - timestamp_origin.

The result is an estimate: it does not indicate whether the Location exists, and
Objects that carry a correction might not be close to the estimate.

# Publisher Restarts {#restarts}

Because Track Properties cannot change, a publisher that restarts and resumes
publishing the same Track cannot revise its origin or re-anchor its mapping; it
MUST reuse already established Track Properties.  A publisher that might
restart SHOULD choose Properties that remain valid and compress well across a
restart.

# Defining Additional Timestamps {#additional-timestamps}

Some applications need more than one timestamp per Object -- for example, a
media mapping that distinguishes presentation time from decode time.  This
document defines a single timestamp per Object; a specification that needs
others can define them as additional Object Properties.  Such a specification
SHOULD designate which of its timestamps is the Object's timestamp defined in
this document, and SHOULD define each additional timestamp as follows:

* Its value is a signed offset from the Object's timestamp, encoded using the
  zig-zag mapping in {{zig-zag}}:

  ~~~
    additional_timestamp = object_timestamp + offset
  ~~~

* The offset is counted in ticks of the Track's Timescale, and the additional
  timestamp is on the same clock, with the same origin, as the Object's
  timestamp.

# IANA Considerations {#iana}

This document registers the following entries in the "MOQ Properties" registry
established by {{MOQT}}.  The code points below are provisional values for
interoperability testing; final values are to be assigned by IANA.  The Object
Property uses a short (two-byte) code point because it is sent per Object.

## TIMESCALE Property

| Type | Name | Scope | Specification |
|-----:|:-----|:------|:--------------|
| 0x2C7A51E0 | TIMESCALE | Track | This document, {{timescale}} |

The value is a variable-length integer giving ticks per second.

## CLOCK_ID Property

| Type | Name | Scope | Specification |
|-----:|:-----|:------|:--------------|
| 0x3E8D2B70 | CLOCK_ID | Track | This document, {{clock-id}} |

The value is a variable-length integer identifying the Track's clock; 0
identifies wall-clock time, and other values identify shared clocks.

## TIMESTAMP_ORIGIN Property

| Type | Name | Scope | Specification |
|-----:|:-----|:------|:--------------|
| 0x31B49A6E | TIMESTAMP_ORIGIN | Track | This document, {{timestamp-origin}} |

The value is a variable-length integer giving the position, in ticks on the
Track's clock, that corresponds to a timestamp of 0.

## TIMESTAMP_MAPPING Property

| Type | Name | Scope | Specification |
|-----:|:-----|:------|:--------------|
| 0x27F308C5 | TIMESTAMP_MAPPING | Track | This document, {{timestamp-mapping}} |

The value is four variable-length integers: a Base Group, and a Base Timestamp,
Group Multiplier, and Object Multiplier, each in ticks.

## OBJECT_TIMESTAMP Property

| Type | Name | Scope | Specification |
|-----:|:-----|:------|:--------------|
| 0x2D1A | OBJECT_TIMESTAMP | Object | This document, {{object-timestamp}} |

The value is a variable-length integer giving the Object's timestamp in ticks
or, on a Track with a Timestamp Mapping, a zig-zag encoded correction to the
Object's mapped timestamp.

# Security Considerations

Timestamps are supplied by the publisher and are not authenticated by the
transport.  An endpoint that acts on timestamps (for buffering, ordering, or
expiry) SHOULD treat them as hints and apply its own sanity checks, since a
misbehaving publisher can send misleading values.

Timestamps and the Timestamp Origin can reveal information about the publisher's
clock and the temporal structure of its content.  Where this is sensitive, a
publisher MAY omit CLOCK_ID (and hence the origin), use a coarser Timescale, or
omit these Properties.  An end-to-end encrypted payload can carry timing that is
hidden from Relays, but when a Timestamp Mapping is in use, Group IDs and Object
IDs reveal timing regardless.  See {{MOQT}} for general considerations on
logging untrusted Property values.

--- back

# Examples {#examples}

The following examples show how common timing arrangements, including those of
the specifications in {{related}}, are expressed with the Properties in this
document.

## Fixed Cadence

A Track sends 30000/1001 Objects per second in Groups of 60 Objects, with
sequential Group IDs starting at 0, and the publisher knows the wall-clock time
W (in ticks since the Unix epoch) at which Group 0 starts:

~~~
  TIMESCALE         = 30000
  CLOCK_ID          = 0
  TIMESTAMP_ORIGIN  = W
  TIMESTAMP_MAPPING = (0, 0, 60060, 1001)
~~~

Objects on cadence carry no timestamp Property; an Object that deviates from the
cadence carries a small correction in OBJECT_TIMESTAMP.

## Explicit Timestamps

A Track whose Objects each carry a wall-clock timestamp in microseconds, with
its origin at 2026-01-01T00:00:00Z:

~~~
  TIMESCALE         = 1000000
  CLOCK_ID          = 0
  TIMESTAMP_ORIGIN  = 1767225600000000
~~~

Each Object carries OBJECT_TIMESTAMP, counted in microseconds since the origin.
Omitting TIMESTAMP_ORIGIN instead gives microseconds since the Unix epoch, as
LOC does when no Timescale is present {{LOC}}, at the cost of larger values.

## Group IDs as Timestamps

A Track whose Group IDs are microseconds since the Unix epoch and whose Objects
share their Group's time, such as an MSF log track {{MSF}}:

~~~
  TIMESCALE         = 1000000
  CLOCK_ID          = 0
  TIMESTAMP_MAPPING = (0, 0, 1, 0)
~~~

Every Object's timestamp is its Group ID, with no per-Object bytes.

## Timeline Template

An MSF timeline template {{MSF}} with a start media time M, start Location (G,
0), Location delta (1, 0), and start wall-clock time W, in which the media time
and wall-clock deltas are both D, all in milliseconds, is expressed as:

~~~
  TIMESCALE         = 1000
  CLOCK_ID          = 0
  TIMESTAMP_ORIGIN  = W - M
  TIMESTAMP_MAPPING = (G, M, D, 0)
~~~

This requires a known wall-clock time with W at least M.  For on-demand content,
where MSF sets the wall-clock values to 0, the publisher omits CLOCK_ID and
TIMESTAMP_ORIGIN, or uses a shared Clock ID as in {{shared-clock-example}}.  The
template describes only Group start times; a publisher that also knows its
per-Object cadence sets the Object Multiplier accordingly.

## Shared Clock Without Wall-Clock Time {#shared-clock-example}

The audio and video Tracks of an on-demand asset have no meaningful wall-clock
time but need to be aligned.  The publisher derives a Clock ID for the asset,
for example from a hash of its identifier; both Tracks carry it and start at
time 0 on that clock:

~~~
  Video:  TIMESCALE = 90000, CLOCK_ID = 0x1A3F5C9E07B2D461
  Audio:  TIMESCALE = 48000, CLOCK_ID = 0x1A3F5C9E07B2D461
~~~

A receiver aligns an audio Object and a video Object by comparing their
timestamps in seconds.

# Acknowledgments
{:numbered="false"}

The authors thank the authors of {{LOC}}, {{MSF}}, and {{TIMESTAMP-LCURLEY}},
whose timestamp work ({{related}}) this document builds on, and the participants
in the MOQ working group discussions that shaped it.

Portions of this document were drafted with the assistance of Claude (Claude
Code, Anthropic).
