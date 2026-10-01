---
title: "Subscriber-Side Multi-Variant Fast Switching for Media over QUIC"
abbrev: moq-fast-switching
docname: draft-ugur-moq-fast-switching-latest
category: info

ipr: trust200902
area: "Applications and Real-Time"
submissiontype: IETF
keyword:
 - media over quic
 - fast switching
 - adaptive bitrate
 - GOP ladder
venue:
  mail: "moq@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/moq/"
  github: "moqtail/draft-ugur-moq-fast-switching"
  latest: "https://moqtail.github.io/draft-ugur-moq-fast-switching/draft-ugur-moq-fast-switching.html"

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs, docmapping]

author:
 -
    ins: D. Ugur
    name: Deniz Ugur
    organization: Ozyegin University
    email: deniz.ugur@ozu.edu.tr
 -
    ins: Z. Gurel
    name: Zafer Gurel
    organization: Ozyegin University
    email: zafer.gurel@ozu.edu.tr
 -
    ins: A. C. Begen
    name: Ali C. Begen
    organization: Ozyegin University
    email: ali.begen@ozyegin.edu.tr

normative:
  MoQTransport: I-D.ietf-moq-transport
  MSF: I-D.ietf-moq-msf

informative:
  SWITCH-FROM-PR:
    title: "Track Switching via the SWITCH_FROM parameter"
    date: 2026
    target: "https://github.com/moq-wg/moq-transport/pull/1674"
  SWITCH-FROM-SOFT-PR:
    title: "SWITCH_FROM Soft Mode"
    date: 2026
    target: "https://github.com/moq-wg/moq-transport/pull/1675"
  MSF-ZAPPING-PR:
    title: "initial text on zapping"
    date: 2026
    target: "https://github.com/moq-wg/msf/pull/122"

--- abstract

This document describes Subscriber-Side Fast Switching (SFS), a technique
that allows a Media over QUIC Transport (MOQT) subscriber to switch between
tracks that carry the same encoded quality but use different Group of
Pictures (GOP) lengths, combining the compression efficiency of a long GOP
with the low switching latency of a short GOP. A publisher using this
technique packages each encoded quality as a ladder of tracks distinguished
only by their keyframe interval, numbers those tracks' Group IDs so that they
share a common alignment, and advertises the keyframe interval of each track
in the MOQT Streaming Format (MSF) catalog. A subscriber uses this
information to compute the nearest valid switch point on any track in scope
and to perform the switch using the SWITCH_FROM request parameter in Soft
mode.

--- middle

# Introduction

Adaptive bitrate (ABR) delivery of live and on-demand media involves a
tension between two goals. Long GOPs improve compression efficiency, since
inter-coded pictures are cheaper to send than keyframes. Short GOPs reduce
the latency of joining a track or switching to a different one, since a
subscriber can only begin decoding a track at a keyframe. A publisher
therefore has to pick a single GOP length that compromises between these two
goals for every track it offers, unless more than one GOP length is made
available at once.

This document describes a technique, Subscriber-Side Fast Switching (SFS),
in which a publisher offers several tracks per encoded quality, each using a
different GOP length, and a subscriber picks whichever of these tracks gets
it to its desired quality soonest. Because every track offering the same
quality carries identical content, encoded at the same bitrate and
resolution, a subscriber can switch between them purely to shorten its wait
for a keyframe, and can equally switch between tracks of different qualities
by considering the keyframe opportunities of both the current and the target
quality's tracks together.

This technique depends on two other specifications. It reuses the Catalog
and Track Object model, including the "keyframeSpacing" and "altGroup" Track
Object fields, defined by the MOQT Streaming Format {{MSF}}. It performs the
actual switch using the SWITCH_FROM request parameter and its Soft mode,
proposed for {{MoQTransport}} in {{SWITCH-FROM-PR}} and
{{SWITCH-FROM-SOFT-PR}} respectively. Neither pull request had been merged
into {{MoQTransport}} at the time of writing; this document is written as
though both are already part of it, and implementers should consult the
referenced pull requests for their current status before relying on this
document.

An earlier, narrower proposal for MSF, {{MSF-ZAPPING-PR}}, described how a
subscriber can use the "keyframeSpacing" field to compute a single valid
switch point when first joining a track. This document is intended to
supersede that proposal as the primary specification of keyframe-interval-
based switching for MOQT: it keeps the "keyframeSpacing" field but adds the
Group ID alignment convention, the multi-track and multi-quality selection
procedure, and the SWITCH_FROM-based switching procedure needed to use it
continuously, not only when first joining a broadcast.

## Document Structure

{{model}} defines the terms used in this document. {{overview}} gives a
high-level view of the technique. {{publisher-reqs}} specifies how a
publisher packages a set of switchable tracks. {{catalog}} specifies the
Catalog fields a publisher uses to advertise them. {{selection}} specifies
how a subscriber computes candidate switch points and selects among them.
{{switching}} specifies how a subscriber executes a switch using SWITCH_FROM.
{{buffering}} describes the buffering a subscriber needs during a switch.
{{security}} and {{iana}} close the document with Security and IANA
Considerations.

# Conventions and Definitions {#model}

{::boilerplate bcp14-tagged}

The following terms are used with the first letter capitalized:

Quality:
: A single encoded representation of media content, identified by properties
  such as resolution, bitrate, and codec profile, excluding its keyframe
  interval.

GOP:
: Group of Pictures. The set of coded pictures between one keyframe
  (inclusive) and the next (exclusive). Within this document, each MOQT
  Group ({{MoQTransport}}) carries exactly one GOP (see {{publisher-reqs}}).

Switch Set:
: The set of Tracks that share one "altGroup" value ({{MSF}}) and among
  which a subscriber performs SFS. A Switch Set can contain Tracks of more
  than one Quality.

GOP Ladder:
: The subset of a Switch Set's Tracks that share one Quality but differ in
  their "keyframeSpacing" ({{MSF}}) value. A Switch Set contains one GOP
  Ladder per Quality it offers.

Alignment Unit (Gm):
: The smallest "keyframeSpacing" value among all Tracks in a Switch Set.

Frame Number:
: A non-negative integer, comparable across every Track in a Switch Set,
  computed from a Location ({{MoQTransport}}) as described in {{selection}}.

Current Track:
: The Track a subscriber is receiving at the time it evaluates a switch.

Target Track:
: A Track a subscriber is considering switching to.

Switch Point:
: A Group ID on a Target Track at which a subscriber can begin receiving
  that track without missing or duplicating any presented media.

# Overview {#overview}

A publisher using SFS offers one or more Qualities, each as a GOP Ladder of
two or more Tracks that are otherwise identical but use different keyframe
intervals, all combined into a single Switch Set. For example, a publisher
might offer a 720p Quality and a 1080p Quality, each with short-, medium-,
and long-GOP Tracks, for a Switch Set of six Tracks in total:

~~~ascii-figure
Switch Set (altGroup = 1)
 +-- 720p GOP Ladder
 |    +-- 720p-short   (keyframeSpacing = 2)
 |    +-- 720p-medium  (keyframeSpacing = 4)
 |    +-- 720p-long    (keyframeSpacing = 8)
 +-- 1080p GOP Ladder
      +-- 1080p-short  (keyframeSpacing = 2)
      +-- 1080p-medium (keyframeSpacing = 4)
      +-- 1080p-long   (keyframeSpacing = 8)
~~~
{: title="Example Switch Set with two Qualities"}

The publisher numbers the Group IDs of every Track in the Switch Set
according to a single, shared Alignment Unit, so that any Group ID on any
Track corresponds to a specific, predictable position in every other Track
({{publisher-reqs}}), and advertises each Track's "keyframeSpacing" and
"altGroup" in the Catalog ({{catalog}}).

A subscriber that is receiving one Track from the Switch Set continuously
re-evaluates whether a different Track in the same GOP Ladder, or a
different GOP Ladder entirely, has a closer upcoming Switch Point
({{selection}}). When it finds one, it issues a SUBSCRIBE or REQUEST_UPDATE
carrying SWITCH_FROM in Soft mode, targeting that Switch Point
({{switching}}), and buffers objects from both Tracks until the handoff
completes ({{buffering}}).

# Publisher Requirements {#publisher-reqs}

A publisher that offers a Switch Set for use with SFS MUST meet the
following requirements for every Track in that Switch Set, in addition to
any other applicable requirements of {{MSF}}:

* Each MOQT Group MUST begin with exactly one keyframe, and MUST NOT contain
  any other keyframe. This ensures that every Group boundary is a usable
  join or switch point, and that no join or switch point is hidden inside a
  Group.

* Every Track in the Switch Set MUST be time-aligned with every other Track
  at Object boundaries: the media presentation time of the first Object of
  a Group on one Track MUST coincide with the media presentation time of
  some Object on every other Track in the Switch Set.

* Every Track's "keyframeSpacing" value MUST be an integer multiple of the
  Switch Set's Alignment Unit (Gm), i.e., the smallest "keyframeSpacing"
  value among all Tracks in the Switch Set. This is what makes the Frame
  Number defined in {{selection}} comparable across every Track in the
  Switch Set, including Tracks of different Qualities.

* For a Track whose "keyframeSpacing" is S, let k = S / Gm. The publisher
  MUST number that Track's Groups 1, 1 + k, 1 + 2k, 1 + 3k, and so on
  (instead of 1, 2, 3, ...), and MUST include a Prior Group ID Gap Object
  Property ({{MoQTransport}}) with value k - 1 on the first Object of every
  Group after the first. This signals that the k - 1 intervening Group ID
  values are intentionally never used, rather than missing.

For example, a Track with keyframeSpacing 8 in a Switch Set whose Alignment
Unit is 2 has k = 4, and is numbered Group 1, Group 5, Group 9, and so on;
the first Object of Group 5 carries a Prior Group ID Gap of 3, indicating
that Groups 2, 3, and 4 will never be published on this Track.

# Catalog {#catalog}

A publisher advertises the properties needed for SFS using two Track Object
fields already defined by {{MSF}}:

altGroup:
: Tracks that are part of the same Switch Set MUST share one "altGroup"
  value, and MUST NOT share that value with any Track outside the Switch
  Set.

keyframeSpacing:
: Every Track in a Switch Set MUST include "keyframeSpacing", populated with
  the number of Objects in each of that Track's Groups (equivalently, its
  keyframe interval expressed in Objects). A subscriber uses this field, and
  the constraints in {{publisher-reqs}}, to identify each Track's GOP Ladder
  (the other Tracks in the Switch Set that share the same Quality) and to
  compute Frame Numbers and Switch Points as described in {{selection}}.

This document does not define any further Catalog fields; a subscriber
distinguishes Tracks of the same Quality from Tracks of different Qualities
within a Switch Set using the same Quality-identifying Track Object fields
it would otherwise use to select among non-SFS Tracks (for example, "width",
"height", and "bitrate").

# Client-Side Track and Group Selection {#selection}

## Frame Numbers

Within one Switch Set, let Gm be the Alignment Unit. For a Track with
keyframeSpacing S, let k = S / Gm as in {{publisher-reqs}}. The Frame Number
of Object O (0-based, per {{MoQTransport}}) of Group G on that Track is:

~~~
frameNumber = (G - 1) * Gm + O + 1
~~~

Because every Track's Group IDs have been numbered as described in
{{publisher-reqs}}, frameNumber increases by exactly one for each
successive Object across Group boundaries on a single Track, and, by
construction, identifies the same position in the underlying media timeline
on every Track in the Switch Set.

## Switch Points

Given a candidate Target Track with keyframeSpacing T (also a multiple of
Gm, per {{publisher-reqs}}), and the subscriber's current Frame Number F
(computed from the Current Track's most recently received Location), the
nearest Switch Point on the Target Track that is not earlier than F is Group
ID:

~~~
targetGroup(T) = 1 + (T / Gm) * ceil((F - 1) / T)
~~~

This is the Group ID of the first Group on the Target Track whose first
Object's Frame Number is greater than or equal to F. The number of Frame
Numbers between F and that Switch Point,

~~~
Distance(T) = (targetGroup(T) - 1) * Gm + 1 - F
~~~

is zero if F already falls exactly on a Group boundary of the Target Track,
and otherwise gives subscribers a way to compare how soon different
candidate Target Tracks reach a Switch Point.

## Selection Policy

A subscriber evaluates candidate Target Tracks and picks one as follows:

* If the subscriber is not changing Quality, the candidates are the other
  Tracks in the Current Track's GOP Ladder with a strictly larger
  "keyframeSpacing" than the Current Track. The subscriber picks the
  candidate with the smallest Distance, i.e., the one that lets it move to
  a more efficient Track as soon as possible. A subscriber that finds no
  Switch Point closer than remaining on the Current Track SHOULD NOT switch.

* If the subscriber is changing to a different Quality, the candidates are
  every Track in the Target Quality's GOP Ladder, together with the other
  Tracks in the Current Track's GOP Ladder with a strictly larger
  "keyframeSpacing" than the Current Track. The subscriber picks the
  candidate with the smallest Distance. If more than one candidate ties for
  the smallest Distance, the subscriber prefers a candidate in the Target
  Quality's GOP Ladder, since it makes direct progress toward the desired
  Quality. A switch to a Track in the Current Track's own GOP Ladder in this
  case is an intermediate step and does not end the Quality change; the
  subscriber repeats this procedure after each switch until it reaches the
  Target Quality.

A subscriber MAY instead implement a simpler policy that only ever considers
one GOP Ladder at a time (the Current Track's GOP Ladder, or, when changing
Quality, only the Target Quality's GOP Ladder) and that breaks ties by
preferring the candidate with the largest "keyframeSpacing" rather than by
preferring progress toward the Target Quality. This simpler policy is easier
to implement and is already in use in at least one implementation of this
document, at the cost of occasionally taking a less direct path to the
Target Quality.

## Worked Example

Consider the Switch Set from {{overview}}, with Gm = 2. A subscriber is
receiving 720p-long (keyframeSpacing 8, k = 4) and has just received Object
3 of Group 13 (the fourth Group of this Track, since Group 1 is the first
and Groups are numbered 1, 5, 9, 13, ...). Its Frame Number is:

~~~
frameNumber = (13 - 1) * 2 + 3 + 1 = 28
~~~

Suppose the application now requests a switch to the 1080p Quality. The
subscriber computes targetGroup(T) and Distance(T) for every Track in the
1080p GOP Ladder:

| Track        | T | targetGroup(T) | Distance(T) |
|:-------------|--:|---------------:|------------:|
| 1080p-short  | 2 |              15 |           1 |
| 1080p-medium | 4 |              15 |           1 |
| 1080p-long   | 8 |              17 |           5 |

1080p-short and 1080p-medium tie for the smallest Distance; both are in the
Target Quality's GOP Ladder, so the subscriber picks one of them, for
example 1080p-medium, and switches to Group 15 of that Track.

# Performing the Switch {#switching}

A subscriber performs a switch found as described in {{selection}} by
sending a SUBSCRIBE, or a REQUEST_UPDATE on an existing subscription,
carrying the SWITCH_FROM parameter ({{SWITCH-FROM-PR}}) in Soft mode
({{SWITCH-FROM-SOFT-PR}}), with the Switch From Request ID identifying the
Current Track's subscription and the Start Group set to the computed
targetGroup(T).

A subscriber MUST NOT use Hard mode for this purpose. Because the Current
and Target Tracks use different keyframe intervals, a relay has no way to
know, from Hard mode alone, where the two Tracks' Objects correspond to the
same point in the media timeline, and could stop forwarding the Current
Track before the Target Track has anything to show in its place. Soft mode
avoids this by having the publisher continue the Current Track's
subscription up to a computed End Group before stopping it, giving a clean
handoff instead of a gap or an overlap of arbitrary duration.

Per {{SWITCH-FROM-SOFT-PR}}, Soft mode stops the suspending subscription at
"Start Group - 1". Because Group IDs are not comparable 1:1 across Tracks
with different "keyframeSpacing" values, a subscriber instead computes the
End Group it expects the publisher to use directly from the Current Track's
own numbering. Let S and k_old = S / Gm be the Current Track's own
keyframeSpacing and group increment, and let switchFrame be the Frame
Number of the chosen Switch Point, (targetGroup(T) - 1) * Gm + 1:

* If the Target Track's keyframeSpacing T is greater than S, the Switch
  Point can fall after the Group the Current Track is presently sending, so
  the expected End Group is the Current Track's own Group whose last Frame
  Number is the greatest multiple of S not exceeding switchFrame - 1:

  ~~~
  End Group = 1 + k_old * (floor((switchFrame - 1) / S) - 1)
  ~~~

  For example, in a Switch Set with Gm = 2, a Current Track with S = 4
  (k_old = 2) sending Frame Number 9 (its Group 5), switching to a Target
  Track with T = 6 and switchFrame = 13, has End Group
  1 + 2 * (floor(12 / 4) - 1) = 1 + 2 * 2 = 5: the Current Track's Group 5
  is also its last Group before the switch.

* Otherwise (T is less than or equal to S), the Current Track has no Group
  boundary of its own between the Group it is presently sending and the
  Switch Point, so the expected End Group is simply the Current Track's own
  Group ID at the time of the switch; the subscriber cannot cut over any
  earlier than the end of the Group it is already receiving. Continuing the
  worked example in {{selection}}, switching from 720p-long (S = 8) to
  1080p-medium (T = 4) has T less than S, so the expected End Group is
  Group 13, the Group the subscriber was already receiving.

A subscriber does not need to separately unsubscribe from the Current
Track's subscription: once the publisher has delivered the End Group, it
concludes the suspending subscription with PUBLISH_DONE.

A subscriber SHOULD subscribe in advance to Tracks it may switch to, using
the FORWARD parameter ({{MoQTransport}}) set to 0, before it expects to need
them. This gives the publisher, or a relay on the path, the opportunity to
establish or warm any upstream state for those Tracks ahead of time, so that
the switch computed in {{selection}} is not delayed by the time it takes to
establish a new subscription. A subscriber SHOULD bound the number of Tracks
it keeps subscribed in this way at once; see {{security}}.

# Client Buffering and Playout Continuity {#buffering}

Because Soft mode keeps delivering the Current Track up to its End Group
while the Target Track begins delivering from its Switch Point, a
subscriber receives Objects from both Tracks for a short overlap. A
subscriber MUST hold received Objects from both Tracks in a small buffer,
keyed by Frame Number, and release them for presentation in Frame Number
order, so that the presented media neither skips nor repeats any frame
across the transition. A buffer sized to hold slightly more than one
Alignment Unit's worth of Objects from each of the two Tracks involved in a
switch is sufficient for this purpose; a subscriber does not need to retain
more than this once a switch has completed.

# Security Considerations {#security}

A subscriber that follows the recommendation in {{switching}} to
pre-establish subscriptions to candidate Target Tracks ahead of time
increases the number of subscriptions, and therefore the amount of upstream
state and bandwidth, a publisher or relay must maintain on its behalf. A
subscriber MUST bound the number of Tracks it keeps subscribed with FORWARD
set to 0 for this purpose, and a publisher or relay MAY apply the limits
described in {{MoQTransport}} to protect itself from excessive resource
consumption by a subscriber that does not.

The set of Tracks and their "keyframeSpacing" values in a Switch Set reveal
the shape of a publisher's ABR ladder to anyone who can observe the Catalog,
in the same way that the set of bitrates and resolutions already does; this
document does not introduce any new information disclosure beyond what
{{MSF}} and {{MoQTransport}} already describe.

# IANA Considerations {#iana}

This document has no IANA actions. It reuses the "keyframeSpacing" and
"altGroup" Track Object fields defined by {{MSF}} and the Prior Group ID Gap
Object Property, the FORWARD parameter, and the SWITCH_FROM parameter
defined or proposed for {{MoQTransport}}, without modifying or extending any
of them.

# Acknowledgments
{:numbered="false"}

The authors thank the participants of the Media Over QUIC Working Group,
and in particular the authors of and commenters on {{SWITCH-FROM-PR}},
{{SWITCH-FROM-SOFT-PR}}, and {{MSF-ZAPPING-PR}}, whose discussion of track
switching and keyframe-interval-based switch points informed this document.

# Use of Generative AI
{:numbered="false"}

Generative AI tools were used to assist with drafting and editing text for
this document. All AI-generated content was reviewed and approved by the
editors.

--- back

# Change Log
{:numbered="false" removeInRFC="true"}

RFC Editor's Note: Please remove this section prior to publication of a
final version of this document.

## Since initial version
{:numbered="false"}

* Initial version.
