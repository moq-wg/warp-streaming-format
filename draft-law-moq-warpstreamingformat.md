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
  MoQTransport: I-D.draft-ietf-moq-transport-05
  CMAFpackaging: I-D.draft-wilaw-moq-cmafpackaging-01
  LOC: I-D.draft-mzanaty-moq-loc-03
  COMMON-CATALOG-FORMAT: I-D.draft-ietf-moq-catalogformat-01
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

WARP Streaming Format (WARP) is a media format designed to deliver CMAF {{CMAF}} and LOC {{LOC}} compliant media content over Media Over QUIC Transport (MOQT) {{MoQTransport}}. WARP works by fragmenting the bitstream into objects that can be independently transmitted. WARP leverages the Common Catalog Format {{COMMON-CATALOG-FORMAT}} to describe the output of the original publisher. WARP specifies how content should be packaged and signaled, defines how the catalog communicates the content, specifies prioritization strategies for real-time and workflows for beginning and terminating broadcasts. WARP also details how end-subscribers may perform adaptive bitrate switching. WARP is targeted at real-time and interactive levels of live latency.

This document describes version 1 of the streaming format.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the conventions detailed in Section 1.3 of {{!RFC9000}} when describing the binary encoding.

# Media packaging {#mediapackaging}
WARP delivers CMAF {{CMAF}} and LOC {{LOC}} packaged media bitstreams. Either format may be used in a broadcast, or they may be intermixed between tracks. The packaging format of a track, once declared, MUST remain constant.

## CMAF packaging
This specification references {{CMAFpackaging}} to define how CMAF packaged bitstreams are mapped to {{MoQTransport}} groups and objects.

Both CMAF Object mappings {{CMAFpackaging}} Section 4 are supported and a content producer may use either. To identify to clients which object mapping mode is being used for a given Track, the catalog "packaging" field MUST use one of the values defined in Table 1. The values are case-sensitive.

Table 1 provides values for the catalog "packaging" field with CMAF packaging.

| Packaging field value     |  Condition                         |                                              Explanation                                            |
|:==========================|:===================================|:====================================================================================================|
| cmaf-frag-per-group       | {{CMAFpackaging}} 4.1 is active    |  Each CMAF Fragment is placed in a single MOQT Object and there is one MOQT Object per MOQT Group   |
| cmaf-chunk-per-object     | {{CMAFpackaging}} 4.2 is active    |  Each CMAF chunk is placed in a MOQT Object and there is one MOQT Group per CMAF Fragment           |

## LOC packaging
This specification references Low Overhead Container (LOC) {{LOC}} to define how audio and video content is packaged. With this packaging mode, each EncodedAudioChunk or EncodedVideoChunk sample is placed in a separate MOQT Object. Samples that belong to the same Group of Pictures (GOP) MUST be placed within the same MOQT Group.

Table 2 provides values for the catalog "packaging" field with LOC packaging.

| Packaging field value     |  Condition                    |                                              Explanation                                    |
|:==========================|:==============================|:============================================================================================|
| loc                       | {{LOC}} packagin is active    |  Each EncodedAudioChunk or EncodedVideoChunk sample is placed in a separate MOQT Object     |


## Time-alignment {#timealignment}
WARP Tracks MAY be time-aligned. Those that are, are subject to the following requirements:

* Time-aligned tracks MUST be advertised in the catalog as belonging to a common render group.
* The presentation time of the first media sample contained within the first MOQT Object of each equally numbered MOQT Group MUST be identical.

A consequence of this restriction is that a WARP receiver SHOULD be able to cleanly switch between time-aligned media tracks at group boundaries.

## Content protection and encryption {#contentprotection}

The catalog and media object payloads MAY be encrypted. Common Encryption {{CENC}} with 'cbcs' mode (AES CBC with pattern encryption) is the RECOMMENDED encryption method for CMAF packaged content.

ToDo - details of how keys are exchanged and license servers signaled. May be best to extend catalog spec to allow the specification of content protection schema, along with any pssh or protection initialization data.

ToDo - content protection for LOC-packaged content.

# Catalog

WARP uses the Common Catalog Format {[COMMON-CATALOG-FORMAT}} to describe the content being produced by a publisher.

Per Sect 5.1 of {{COMMON-CATALOG-FORMAT}}, WARP registers an entry in the "MoQ Streaming Format Type" table.  The type value is 0x001, the name is "WARP Streaming Format" and the RFC is XXX.

Every WARP catalog MUST declare a streaming format type (See Sect 3.2.1 of {{COMMON-CATALOG-FORMAT}}) value of 1.

Every WARP catalog MUST declare a streaming format version (See Sect 3.2.1 of {{COMMON-CATALOG-FORMAT}}) corresponding to the version of this document.

The catalog track MUST have a track name of "catalog". A catalog object MAY be independent of other catalog objects or it MAY represent a delta update of a prior catalog object. The first catalog object published within a new group MUST be independent.  A catalog object SHOULD only be published only when the availability of tracks changes.

Each catalog update MUST be mapped to a discreet MOQT Object.

# Media transmission
The MOQT Groups and MOQT Objects need to be mapped to MOQT Streams. Irrespective of the {{packagingmode}} in place, each MOQT Object MUST be mapped to a new MOQT Stream.

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
