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
    fullname: Kirill Pugin
    organization: Meta
    email: ikir@meta.com
 -
    fullname: Suhas Nandakumar
    organization: Cisco
    email: snandaku@cisco.com
 -
    fullname: Victor Vasiliev
    organization: Google
    email: vasilvv@google.com

normative:
  QUIC: RFC9000
  ISOBMFF:
    title: "Information technology — Coding of audio-visual objects — Part 12: ISO Base Media File Format"
    date: 2015-12
  CMAF:
    title: "Information technology -- Multimedia application format (MPEG-A) -- Part 19: Common media application format (CMAF) for segmented media"
    date: 2020-03
  MoQTransport: I-D.lcurley-warp


informative:


--- abstract

This document specifies the WARP Media Format, designed to operate on MoQTransport.


--- middle

# Introduction

WARP Media Format (WMF) is a media format designed to deliver CMAF {{CMAF}} compliant media content over MoQTransport {{MoQTransport}}. WMF  works by fragmenting the bitstream into objects that can be transmitted independently. WMF leverages a simple prioritization strategy of assigning newer content a higher send priority, allowing intermediaries to drop older data, and video over audio, in the face of congestion. Complete Groups of Pictures (GOPS) {{ISOBMFF}} are mapped to MoQ transport Objects. WMF is targeted at interactive levels of live latency.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the conventions detailed in Section 1.3 of {{!RFC9000}} when describing the binary encoding.


# Catalog format

The format of the CATALOG payload, as defined by {{MoQTransport}} Sect X.X,  is as follows:

~~~
CATALOG payload {
  media format type (i)
  track count (i),
  track descriptors (..)
}
~~~
{: #warpmedia-catalog-body title="WARP Media Format CATALOG body"}

* Media format type: this MUST hold the value 0x001 (see {{IANA Considerations}}).

* Track count:
The number of tracks described by the catalog. A CATALOG MAY hold 0 tracks.

Each track is described by a track descriptor with the format:

~~~
Track Descriptor {
  track ID (i),
  init length (i)
  init payload (..)
}
~~~
{: #warpmedia-track-descriptor title="Warp Media Format track descriptor"}

* Track ID:
Within WMF, track IDs are numeric integers. Track IDs SHOULD start at 0 and SHOULD increment by 1 for each additional track.

* Init payload:
The init payload in a track descriptor MUST consist of a File Type Box (ftyp) followed by a Movie Box (moov). This Movie Box (moov) consists of Movie Header Boxes (mvhd), Track Header Boxes (tkhd), Track Boxes (trak), followed by a final Movie Extends Box (mvex). These boxes MUST NOT contain any samples and MUST have a duration of zero. A Common Media Application Format Header {{CMAF}} meets all these requirements.

# Packaging

## Tracks

Each codec bitstream MUST be packaged in to a sequence of Objects within a separate track.

## Objects

Object Delivery Order MUST match the Object sequence number.

The Object payload:

* MUST consist of a Segment Type Box (styp) followed by any number of media fragments. Each media fragment consists of a Movie Fragment Box (moof) followed by a Media Data Box (mdat). The Media Fragment Box (moof) MUST contain a Movie Fragment Header Box (mfhd) and Track Box (trak) with a Track ID (`track_ID`) matching a Track Box in the initialization fragment.
* MUST contain a single track.
* MUST contain media content encoded in decode order. This implies an increasing DTS.
* MAY contain any number of frames/samples. It is RECOMMENDED that each media fragment consists of a single frame to minimize latency.
* MAY have gaps between frames/samples.
* MAY overlap with other objects. This means timestamps may be interleaved between objects.

A Common Media Application Format Segment {{CMAF}} meets all these requirements and is RECOMMENDED as the preferred packaging format.


# Security Considerations

The Object payload MAY be encrypted. CENC Encoding with cbcs cipher mode is RECOMMENDED.


# IANA Considerations

This document has no IANA actions.




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
