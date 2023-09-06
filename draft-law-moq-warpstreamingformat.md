---
title: "WARP Streaming Format"
category: info

docname: draft-law-moq-warpstreamingformat-latest
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
 - WARP
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
    email: wilaw@akamai.com
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
  MoQTransport: I-D.lcurley-moq-transport
  RFC9000: RFC9000
  COMMON-CATALOG-FORMAT:
    title: "Common Catalog Format for moq-transport"
    date: September 6, 2023
    target: https://wilaw.github.io/catalog-format/draft-wilaw-moq-catalogformat.html
  ISOBMFF:
    title: "Information technology -- Coding of audio-visual objects -- Part 12: ISO Base Media File Format"
    date: 2015-12
  CMAF:
    title: "Information technology -- Multimedia application format (MPEG-A) -- Part 19: Common media application format (CMAF) for segmented media"
    date: 2020-03
  CENC:
    title: "International Organization for Standardization - Information technology - MPEG systems technologies - Part 7: Common encryption in ISO base media file format files"
    date: 2020-12

informative:


--- abstract

This document specifies the WARP Streaming Format, designed to operate on Media Over QUIC Transport.


--- middle

# Introduction

WARP Streaming Format (WARP) is a media format designed to deliver CMAF {{CMAF}} compliant media content over Media Over QUIC Transport (MOQT) {{MoQTransport}}. WARP works by fragmenting the bitstream into objects that can be independently transmitted. WARP leverages a simple prioritization strategy of assigning newer content a higher delivery order, allowing intermediaries to drop older data, and video over audio, in the face of congestion. Either complete Groups of Pictures (GOPS) {{ISOBMFF}} or individual frames are mapped to MoQTransport Objects. WARP is targeted at interactive levels of live latency.

This document describes version 1 of the streaming format.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the conventions detailed in Section 1.3 of {{!RFC9000}} when describing the binary encoding.

# Packaging

Each codec bitstream MUST be packaged in to a sequence of Objects within a separate track.

Media tracks SHOULD be media-time aligned. CMAF {{CMAF}} Aligned Switching Sets meet this requirement. A receiver SHOULD be able to cleanly switch between media tracks at group boundaries.

Each group MUST be independently decodeable. Assigning a new group ID to each CMAF Fragment (see {{CMAF}} Sect 6.6.1) meets this requirement.

# Catalog

WARP uses the Common Catalog Format {[COMMON-CATALOG-FORMAT}} to describe the content being produced by a publisher. 

Per Sect 5.1 of {{COMMON-CATALOG-FORMAT}}, WARP registers an entry in the "MoQ Streaming Format Type" table.  The type value is 0x001, the name is "WARP Streaming Format" and the RFC is XXX.

Every WARP catalog MUST declare a streaming format type (See Sect 3.2.1 of {{COMMON-CATALOG-FORMAT}}) value of 1.

Every WARP catalog MUST declare a streaming format version (See Sect 3.2.1 of {{COMMON-CATALOG-FORMAT}}) corresponding to the version of this document. 

The catalog track MUST have a track name of "catalog". A catalog object MAY be independent of other catalog objects or it MAY represent a delta update of a prior catalog object. The first catalog object published within a new group MUST be independent.  A catalog object SHOULD only be published only when the availability of tracks changes.

Each catalog update MUST be mapped to a discreet moq-transport object. 



## Media Objects

Object Delivery Order MUST match the Object sequence number.

The media object payload:

* MUST consist of a Segment Type Box (styp) followed by any number of media fragments. Each media fragment consists of a Movie Fragment Box (moof) followed by a Media Data Box (mdat). The Media Fragment Box (moof) MUST contain a Movie Fragment Header Box (mfhd) and Track Box (trak) with a Track ID (`track_ID`) matching a Track Box in the initialization fragment.
* MUST contain a single {{ISOBMFF}} track.
* MUST contain media content encoded in decode order. This implies an increasing decoding time stamp (DTS).
* MAY contain any number of frames/samples.
* MAY have gaps between frames/samples.
* MAY overlap with other objects. This means timestamps may be interleaved between objects.

Two options are RECOMMENDED for packaging CMAF content into WARP media objects:

* the first is to package a complete CMAF Fragment (see {{CMAF}} sect 6.6.1) into a single object within each group. This results in there being a single GOP (Group of Pictures) in the media object and a single media object per group.
* The second is to package a CMAF chunk (see {{CMAF}} sect 6.6.5), in which the mdat holds a single frame of video, or sample of audio, into each object and to assign a unique group ID to each fragment. This approach is RECOMMENDED to minimize latency.

# Workflow

A WARP publisher MUST publish a catalog track object before publishing any media track objects.

At the completion of a session, a publisher should publish a catalog object with track count of 0. This SHOULD be interpreted by receivers that the publish session is complete.

# Content protection and encryption

The catalog and media object payloads MAY be encrypted. Common Encryption {{CENC}} with 'cbcs' mode (AES CBC with pattern encryption) is the RECOMMENDED encryption method.

ToDo - details of how keys are exchanged and license servers signalled.

# Security Considerations

ToDo

# IANA Considerations {#IANA}

This document creates a new entry in the "MoQ Streaming Format" Registry (see {{MoQTransport}} Sect 8).  The type value is 0x001, the name is "WARP Streaming Format" and the RFC is XXX.

--- back

# Acknowledgments
{:numbered="false"}

- Alan Frindell
- Ali Begen
- Charles Krasic
- Christian Huitema
- Cullen Jennings
- Hang Shi
- James Hurley
- Jordi Cenzano
- Mike English
- the MoQ Workgroup and mailing lists.
