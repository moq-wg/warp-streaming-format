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

# Packaging

Each codec bitstream MUST be packaged in to a sequence of Objects within a separate track.

Media tracks SHOULD be media-time aligned. CMAF {{CMAF}} Aligned Switching Sets meet this requirement. A receiver SHOULD be able to cleanly switch between media tracks at group boundaries.

Each group MUST be independently decodeable. Assigning a new group ID to each CMAF Fragment (see {{CMAF}} sect 6.6.1) meets this requirement.

## Catalog objects

Per {{MoQTransport}} sect X.X, the catalog object MUST have a track name of "catalog".

A catalog object MAY be independent of other catalog objects or it MAY represent a delta update of a prior catalog object. The first catalog object published within a new group MUST be independent.  A catalog object SHOULD only be published only when the availability of tracks changes.

The format of the CATALOG object payload is as follows:

~~~
CATALOG payload {
  media format type (i),
  version (i),
  parent object ID (i),
  track change count (i),
  track change descriptors (..)
}
~~~
{: #warpmedia-catalog-body title="WARP Media Format CATALOG body"}

* Media format type: this MUST hold the value 0x001 (see {{IANA}}). This value MUST NOT be encrypted. 

* Version: this MUST be the version of WMF to which the media packaging and catalog serialization conforms.

* Parent object ID: 0 if this object represents an independent catalog or the parent object ID if this represents a delta update. 

* Track change count:
The number of track changes described by the catalog. A catalog update describing 0 tracks, or deleting all existing tracks, SHALL be interpreted by the WMF client to mean that the publishing session is complete. A WMF client SHOULD process all changes before making a subscription selection. 

Each track change is described by a track change descriptor with the format:

~~~
Track Change Descriptor {
  track name length (i),
  track name (..),
  operation (1),
  change payload(..)
}
~~~
{: #warpmedia-track-descriptor title="Track change descriptor"}
* Track name length: the length of track name field
* Track name: the UTF-8 encoded track name. Within {{MoQTransport}} track names are strings. Track names MUST never be reused. If a track is published and then unpublished, it must be allocated a new track name before it is re-published. A catalog MUST NOT reference itself i.e the the track name must not be "catalog". 
* Operation: a binary flag. 1 if the track is being added and 0 if it is being deleted. A publisher MUST NOT signal deletion of a track that has not been previously added. 
* Change payload:  depends upon the value of the operation flag. If the operation is a 1 (add), then it SHALL hold an Initialization Header. If the operation is 0 (delete), then it SHALL hold a Deletion Header.

~~~
Initialization Header {
  init length (i)
  init payload (..)
}
~~~
{: #warpmedia-initialization-header title="Initialization Header"}
* Init length: the length of the init payload
* Init payload:
The init payload MUST consist of a File Type Box (ftyp) followed by a Movie Box (moov). This Movie Box (moov) consists of Movie Header Boxes (mvhd), Track Header Boxes (tkhd), Track Boxes (trak), followed by a final Movie Extends Box (mvex). These boxes MUST NOT contain any samples and MUST have a duration of zero. A Common Media Application Format Header {{CMAF}} meets all these requirements.

~~~
Deletion Header {
  Last group: (i),
  Last object: (i)
}
~~~
{: #warpmedia-deletion-header title="Deletion Header"}
* Last group: holds the last {{MoQTransport}} Group sequence number published under that track name.
* Last object: holds the last {{MoQTransport}} Object sequence number published under that track name.


## Media Objects

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

This document creates a new entry in the "MoQTransport Media Format" Registry. The type value is 0x001, the name is "WARP Media" and the RFC is XXX.

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
