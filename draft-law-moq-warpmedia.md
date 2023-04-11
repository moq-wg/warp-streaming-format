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
  CENC:
    title: "International Organization for Standardization - Information technology - MPEG systems technologies - Part 7: Common encryption in ISO base media file format files"
    date: 2020-12

informative:


--- abstract

This document specifies the WARP Media Format, designed to operate on MoQTransport.


--- middle

# Introduction

WARP Media Format (WMF) is a media format designed to deliver CMAF {{CMAF}} compliant media content over MoQTransport {{MoQTransport}}. WMF works by fragmenting the bitstream into objects that can be transmitted independently. WMF leverages a simple prioritization strategy of assigning newer content a higher delivery order, allowing intermediaries to drop older data, and video over audio, in the face of congestion. Either complete Groups of Pictures (GOPS) {{ISOBMFF}} or individual frames are mapped to MoQTransport Objects. WMF is targeted at interactive levels of live latency.

This document describes version 1 of the media format.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the conventions detailed in Section 1.3 of {{!RFC9000}} when describing the binary encoding.

# Track Formats {#formats}
This document creates a registry of track formats.
The track format indicates how media is broken into groups and how each object is encoded.

DISCUSS What about non-media formats? Should this be in a separate draft?

|--------------|---------------------------------|
|  Format      |  Name                           |
|-------------:|:--------------------------------|
|  0x0         |  Reserved                       |
|--------------|---------------------------------|
|  0x1         |  Reserved                       |
|--------------|---------------------------------|
|  0xff000000  |  CMAF Catalog {{cmaf-catalog}}  |
|--------------|---------------------------------|
|  0xff000001  |  CMAF Payload {{cmaf-payload}}  |
|--------------|---------------------------------|

This draft defines two track formats, prefixed with 0xff000000.
The prefix will be removed in the final version of the draft.

DISCUSS Should the track format be on the wire? It would have to be at the start of every group.

# Packaging

Each codec bitstream MUST be packaged in to a sequence of Objects within a separate track.

Media tracks SHOULD be media-time aligned. CMAF {{CMAF}} Aligned Switching Sets meet this requirement. A receiver SHOULD be able to cleanly switch between media tracks at group boundaries.

Each group MUST be independently decodeable. Assigning a new group ID to each CMAF Fragment (see {{CMAF}} sect 6.6.1) meets this requirement.

## Catalog objects {#cmaf-catalog}

The catalog object MUST have a track ID of 0.

Each catalog object MUST be independent of other catalog objects and MUST carry a unqiue group sequence number (see {{MoQTransport}}, Sect X.X). The first catalog published MUST have a group sequence number of 0. Every catalog object MUST have an object sequence number of 0 and there MUST be only one object per catalog group. A catalog track object SHOULD be published only when the availability of tracks changes.

The format of the CATALOG object payload, as defined by {{MoQTransport}} Sect X.X,  is as follows:

~~~
CATALOG payload {
  media format type (i),
  version (i),
  track count (i),
  track descriptors (..)
}
~~~
{: #warpmedia-catalog-body title="WARP Media Format CATALOG body"}

* Media format type: this MUST hold the value 0xff000000 (see {{formats}}).

* Version: this MUST be the version of WMF to which the media packaging and catalog serialization conforms.

* Track count:
The number of tracks described by the catalog. A catalog describing 0 tracks is a signal to the WMF client that the publishing session is complete.

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
Within WMF, track IDs are numeric integers. Track IDs SHOULD start at 0 and SHOULD increment by 1 for each additional track. Track IDs MUST never be reused. If a track is published and then unpublished, it must be allocated a new track ID before it is re-published.

* Init payload:
The init payload in a track descriptor MUST consist of a File Type Box (ftyp) followed by a Movie Box (moov). This Movie Box (moov) consists of Movie Header Boxes (mvhd), Track Header Boxes (tkhd), Track Boxes (trak), followed by a final Movie Extends Box (mvex). These boxes MUST NOT contain any samples and MUST have a duration of zero. A Common Media Application Format Header {{CMAF}} meets all these requirements.


## Media Objects {#cmaf-payload}

Object Delivery Order MUST match the Object sequence number.

The media object payload:

* MUST consist of a Segment Type Box (styp) followed by any number of media fragments. Each media fragment consists of a Movie Fragment Box (moof) followed by a Media Data Box (mdat). The Media Fragment Box (moof) MUST contain a Movie Fragment Header Box (mfhd) and Track Box (trak) with a Track ID (`track_ID`) matching a Track Box in the initialization fragment.
* MUST contain a single {{ISOBMFF}} track.
* MUST contain media content encoded in decode order. This implies an increasing DTS.
* MAY contain any number of frames/samples.
* MAY have gaps between frames/samples.
* MAY overlap with other objects. This means timestamps may be interleaved between objects.

Two options are RECOMMENDED for packaging CMAF content into WMF media objects:

* the first is to package a complete CMAF Fragment (see {{CMAF}} sect 6.6.1) into a single object within each group. This results in there being a single GOP (Group of Pictures) in the media object and a single media object per group.
* The second is to package a CMAF chunk (see {{CMAF}} sect 6.6.5), in which the mdat holds a single frame of video, or sample of audio, into each object and to assign a unique group ID to each fragment. This approach is RECOMMENDED to minimize latency.

# Workflow

A WMF publisher MUST publish a catalog track object before publishing any media track objects.

At the completion of a session, a publisher should publish a catalog object with track count of 0. This SHOULD be interpreted by receivers that the publish session is complete.

# Content proection and encruption

The catalog and media object payloads MAY be encrypted. Common Encryption {{CENC}} with 'cbcs' mode (AES CBC with pattern encryption) is the RECOMMENDED encryption method.

ToDo - details of how keys are exchanged and license servers signalled.

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
