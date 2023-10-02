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
  CMAFpackaging: I-D.draft-wilaw-moq-cmafpackaging
  RFC9000: RFC9000
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

# Media packaging {#mediapackaging}
WARP delivers CMAF-packaged media bitstreams. This specification references {{CMAFpackaging}} to define how CMAF packaged bitstreams are mapped to {{MoQTransport}} groups and objects.  

Both CMAF Object mappings {{CMAFpackaging}} Section 4 are supported and a content producer may use either. To identify to consumers which object mapping mode is being used for a given Track, a new track field is defined as per table 1.

Table 1
| Field                   |  Name                  | Required |  Location |  JSON type |      Definition            |
|:========================|:=======================|:=========|:==========|:===========|:===========================|
| WARP packaging mode     | warp-packaging         |  yes     |   RT      |  String    | See {{packagingmode}}      |


## Packaging mode {#packagingmode}
The packaging mode value is defined by Table 2.

Table 2
| warp-packing field value  |  Condition                         |                                              Explanation                                            |
|:==========================|:===================================|:====================================================================================================|
| frag-per-group            | {{CMAFpackaging}} 4.1 is active    |  Each CMAF Fragment is placed in a single MOQT Object and there is one MOQT Object per MOQT Group   |
| chunk-per-object          | {{CMAFpackaging}} 4.2 is active    |  Each CMAF chunk is placed in a MOQT Object and there is one MOQT Group per CMAF Fragment           |


## Time-alignment {timealignment}
WARP Tracks MAY be time-aligned. Those that are, are subject to the following requirements:

* Time-aligned tracks MUST be advertised in the catalog as belonging to a common render group.
* The presentation time of the first media sample contained within the first MOQT Object of each equally numbered MOQT Group MUST be identical.

A consequence of this restriction is that a WARP receiver SHOULD be able to cleanly switch between time-aligned media tracks at group boundaries.

## Content protection and encryption

The catalog and media object payloads MAY be encrypted. Common Encryption {{CENC}} with 'cbcs' mode (AES CBC with pattern encryption) is the RECOMMENDED encryption method.

ToDo - details of how keys are exchanged and license servers signaled. May be best to extend catalog spec to allow the specification of content protection schema, along with any pssh or protection initialization data. 

## Catalog objects

The catalog object MUST have a track name of "catalog".

A catalog object MAY be independent of other catalog objects or it MAY represent a delta update of a prior catalog object. The first catalog object published within a new group MUST be independent.  A catalog object SHOULD only be published only when the availability of tracks changes.

The format of the CATALOG object payload is as follows:

~~~
CATALOG payload {
  media format type (i),
  version (i),
  parent object sequence (i),
  track change count (i),
  track change descriptors (..)
}
~~~
{: #warpmedia-catalog-body title="WARP CATALOG body"}

* Media format type: this MUST hold the value 0x001 (see {{IANA}}). This value MUST NOT be encrypted.

* Version: this MUST be the version of WARP to which the media packaging and catalog serialization conforms.

* Parent object sequence: 0 if this object represents an independent catalog or the {{MoQTransport}} (Sect 6.2) parent object sequence if this represents a delta update.

* Track change count:
The number of track changes described by the catalog. A catalog update describing 0 tracks, or deleting all existing tracks, SHALL be interpreted by the WARP client to mean that the publishing session is complete. A WARP client SHOULD process all changes before making a subscription selection.

Each track change is described by a track change descriptor with the format:

~~~
Track Change Descriptor {
  full track name length (i),
  full track name (..),
  operation (1),
  change payload(..)
}
~~~
{: #warpmedia-track-descriptor title="Track change descriptor"}
* Full track name length: the length of the full track name field
* Full track name: the Full Track Name as defined by {{MoQTransport}} (Sect 2.3.1). Track names MUST never be reused. If a track is published and then unpublished, it must be allocated a new track name before it is re-published. A catalog MUST NOT reference itself i.e the the track name must not be "catalog".
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


# Media transmission
The MOQT Groups and MOQT Objects need to be mapped to moq-transport Streams. Irrespective of the {{#packagingmode}} in place, each MOQT Object MUST be mapped to a new moq-transport Stream.

# Workflow

A WARP publisher MUST publish a catalog track object before publishing any media track objects.

At the completion of a session, a publisher MUST publish a catalog update that removes all currently active tracks.  This action SHOULD be interpreted by receivers to mean that the publish session is complete.



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
