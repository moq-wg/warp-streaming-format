---
title: "WARP Media Format"
category: info

docname: draft-law-moq-warpmedia-latest
submissiontype: IETF  # also: "independent", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Applications and Real-Time"
workgroup: "Media Over QUIC"
keyword:
 - MoQ
 - MoQTransport
venue:
  group: "Media Over QUIC"
  type: "Working Group"
  mail: "moq@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/moq/"
  github: "wilaw/MoQ"
  latest: "https://wilaw.github.io/MoQ/draft-law-moq-warpmedia.html"

author:
 -
    fullname: Will Law
    organization: Akamai
    email: wilaw@akamai.co
 -
    fullname: Luke Curley
    organization: Twitch
    email: kixelated@gmail.com
 -
    fullname: Victor Vasiliev
    organization: Google
    email: vasilvv@google.com
 -
    fullname: Suhas Nandakumar
    organization: Cisco
    email: snandaku@cisco.com
 -
    fullname: Kirill Pugin
    organization: Meta
    email: ikir@meta.com

normative:
  MoQTransport: I-D.lcurley-warp
  QUIC: RFC9000
  ISOBMFF:
    title: "Information technology — Coding of audio-visual objects — Part 12: ISO Base Media File Format"
    date: 2015-12
  CMAF:
    title: "Information technology -- Multimedia application format (MPEG-A) -- Part 19: Common media application format (CMAF) for segmented media"
    date: 2020-03



informative:
  HLS: RFC8216
  LL-HLS: I-D.pantos-hls-rfc8216bis-07
  DASH:
    title: "Information technology — Dynamic adaptive streaming over HTTP (DASH) — Part 1: Media presentation description and segment formats"
    date: 2022-08

--- abstract

This document specifies the WARP Media Format, designed to operate on MoQTransport.


--- middle

# Introduction

WARP Media Format (WMF) is a media format designed to deliver CMAF {{CMAF}} compliant media content over MoQTransport {{MoQTransport}}. WMF works by fragmenting the bitstream into objects that can be transmitted independently. WMF leverages a simple prioritization strategy of assigning newer content a higher delivery order, allowing intermediaries to drop older data, and video over audio, in the face of congestion. Either complete Groups of Pictures (GOPS) {{ISOBMFF}} or individual frames are mapped to MoQTransport Objects. WMF is targeted at interactive levels of live latency.

This document describes version 1 of the media format.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the conventions detailed in Section 1.3 of {{!RFC9000}} when describing the binary encoding.

# CMAF
The Common Media Application Format {{CMAF}} is a MP4 based {{ISOBMFF}} media container used in HLS/DASH.

* A CMAF Catalog track {{cmaf-catalog}} consists of a CMAF Header.
* A CMAF Payload track {{cmaf-payload}} consists of CMAF Fragments which consist of CMAF Chunks.

## Catalog {#cmaf-catalog}
A CMAF catalog contains information about multiple track.

The consumer can use this information to determine if it should subscribe to a track.
For example, the consumer may wish to subscribe to an english audio track but not a japanese audio track.
Additionally, the catalog contains information on how to subscribe to a track.

A CMAF Catalog MUST consist of a single group and single object, both with sequence 0.
It is not possible to modify the catalog in the current draft.

The CMAF Header {{CMAF}} contains basic information about the media, such as the codec, profile, resolution, sample rate, language, etc.
This information is basic and lacks information about the relationship between tracks.
It is RECOMMENDED to use another catalog format for more advanced functionality (ex. variants).
Note that the `track_id` field {{ISOBMFF}} is unrelated to the transport track ID {{MoQTransport}}.


The object payload contains:

~~~
CMAF Catalog {
  CMAF Track Count (i),
  CMAF Track Descriptors (..)
}
~~~
{: #warpmedia-catalog-body title="Warp CMAF Catalog"}

* Track count:
The number of tracks described by the catalog.


Each track is described by a track descriptor with the format:

~~~
CMAF Track Descriptor {
  Track ID (i),
  Track Format (i),
  CMAF Header (b)
}
~~~
{: #warp-track-descriptor title="Warp CMAF Track Descriptor"}

* Track ID:
A unique identifier for the track within the track bundle.
The consumer uses this Track ID to as the parameter to SUBSCRIBE.

* Track Format:
This MUST be 0xff000001 in the current draft, although future drafts MAY allow multiple versions.

* CMAF Header
A CMAF Header {{CMAF}}.
This consists of a File Type Box (ftyp) followed by a Movie Box (moov).
This Movie Box (moov) consists of Movie Header Boxes (mvhd), Track Header Boxes (tkhd), Track Boxes (trak), followed by a final Movie Extends Box (mvex).
These boxes MUST NOT contain any samples and MUST have a duration of zero.
Note that a HLS and DASH init segment meets these requirements.


DISCUSS do we allow multiple CMAF tracks within a transport track?
DISCUSS do tracks need to be fragmented into groups at similar boundaries?


## Payload {#cmaf-payload}
A CMAF Payload contains media samples in an MP4 container {{ISOBMFF}}.

Each transport object {{MoQTransport}} consists of a CMAF Chunk {{CMAF}}.
A CMAF Chunk consists of a Segment Type Box (styp) followed by any number of media fragments.
Each fragment consists of a Movie Fragment Box (moof) followed by a Media Data Box (mdat).
The Media Fragment Box (moof) MUST contain a Movie Fragment Header Box (mfhd) and Track Box (trak) with a Track ID (`track_ID`) matching a Track Box in the track header.

Each transport group {{MoQTransport}} consists of a CMAF Fragment {{CMAF}}.
A CMAF Fragment is independently decodable and consists of one or more CMAF Chunks, as outlined aboved.

## Compatibility
The primary benefit of CMAF is that it's supported by other streaming protocols like HLS and DASH.
In most instances, the segments can be used as the object payload without modification.

Here's a mapping between some of the terminology used between different protocols:

|---------------------|----------------|----------------|-----------------|
| Warp                | Catalog Object | Payload Group  | Payload Object  |
|--------------------:|:---------------|----------------|-----------------|
| CMAF {{CMAF}}       | CMAF Header    | CMAF Fragment  | CMAF Chunk      |
|---------------------|----------------|----------------|-----------------|
| HLS {{HLS}}         | Init Segment   | Media Segment  | N/A             |
|---------------------|----------------|----------------|-----------------|
| DASH {{DASH}}       | Init Segment   | Media Segment  | N/A             |
|---------------------|----------------|----------------|-----------------|
| LL-HLS {{LL-HLS}}   | Init Segment   | Media Segment  | Partial Segment |
|---------------------|----------------|----------------|-----------------|
| LL-DASH {{DASH}}    | Init Segment   | Media Segment  | Media Chunk     |
|---------------------|----------------|----------------|-----------------|


# Security Considerations

ToDo

# IANA Considerations {#IANA}

TODO

--- back

# Acknowledgments
{:numbered="false"}

- Alan Frindell
- Ali Begen
- Charles Krasic
- Cullen Jennings
- Hang Shi
- James Hurley
- Jordi Cenzano
- Mike English
- the MoQ Workgroup and mailing lists.
