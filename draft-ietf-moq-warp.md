---
title: "WARP Streaming Format"
category: info

docname: draft-ietf-moq-warp-latest
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
  github: "moq-wg/warp-streaming-format"
  latest: "https://moq-wg.github.io/warp-streaming-format/
           draft-ietf-moq-warp.html"

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
  MoQTransport: I-D.draft-ietf-moq-transport-11
  LOC: I-D.draft-mzanaty-moq-loc-05
  BASE64: RFC4648
  JSON: RFC8259
  LANG: RFC5646
  MIME: RFC6838
  RFC9000: RFC9000
  RFC4180: RFC4180
  WEBCODECS-CODEC-REGISTRY:
    title: "WebCodecs Codec Registry"
    date: September 2024
    target: https://www.w3.org/TR/webcodecs-codec-registry/
  WEBVTT:
    title: "World Wide Web Consortium (W3C), WebVTT: The Web Video Text Tracks Format"
    date: April 2019
    target: https://www.w3.org/TR/webvtt1/
  IMSC1:
    title: "W3C, TTML Profiles for Internet Media Subtitles and Captions 1.0 (IMSC1)"
    date: April 2016
    target: https://www.w3.org/TR/ttml-imsc1/

informative:


--- abstract

This document specifies the WARP Streaming Format, designed to operate on Media Over QUIC Transport.


--- middle

# Introduction

WARP Streaming Format (WARP) is a media format designed to deliver LOC {{LOC}}
compliant media content over Media Over QUIC Transport (MOQT) {{MoQTransport}}.
WARP works by fragmenting the bitstream into objects that can be independently
transmitted. WARP leverages a catalog format to describe the output of the
original publisher. WARP specifies how content should be packaged and signaled,
defines how the catalog communicates the content, specifies prioritization
strategies for real-time and workflows for beginning and terminating broadcasts.
WARP also details how end-subscribers may perform adaptive bitrate switching.
WARP is targeted at real-time and interactive levels of live latency.

This document describes version 1 of the streaming format.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the conventions detailed in Section 1.3 of {{RFC9000}} when
describing the binary encoding.

# Scope

The purpose of WARP is to provide an interoperable media streaming format
operating over {{MoQTransport}}. Interoperability implies that:

* An original publisher can package incoming media content into tracks, prepare
  a catalog and annouce the availability of the content to a MOQT relay. Media
  content refers to audio and video data, as well as ancillary data such as
  captions, subtitles, accessibility and other timed-text data.
* A MOQT relay can process the annoucement as well as cache and propagate the
  tracks, both to other relays or to the final subscriber.
* A final subscriber can parse the catalog, request tracks, decode and render
  the received media data.

WARP is intended to provide a format for delivering commercial media content. To
that end, the following features are within scope:

* Video codecs - all codecs supported by {{LOC}}
* Audio codecs  - all audio codecs supported by {{LOC}}
* Catalog track - describes the availability and characteristics of content
  produced by the original publisher.
* Timeline track - describes the relationship between MOQT Group and Object IDs
  to media time.
* Token-based authorization and access control
* Captions + Subtitles - support for {{WEBVTT}} and {{IMSC1}} transmission
* Latency support across multiple regimes (thresholds are informative only and
  describe the delay between the original publisher placing the content on the
  wire and the final subscriber rendering it)
 * Real-time - less than 500ms
 * Interactive - between 500ms and 2500ms
 * Standard  - above 2500ms
 * VOD latency - content that was previously produced, is no longer live and is
   available indefinitely.
* Content encryption
* ABR between time-synced tracks - subscribers may switch between between tracks
  at different quality levels in order to maximize visual or audio quality under
  fconditions of throughput variability.
* Capable of delivering interstitial advertising.
* Logs and analytics management - support for the reporting of client-side QoE
   and relay delivery actions.

Initial verisons of WARP will prioritize basic features necessary to exercise
interoperability across delivery systems. Later versions will add commercially
necessary features.

# Media packaging {#mediapackaging}
WARP delivers LOC {{LOC}} packaged media bitstreams.

## LOC packaging
This specification references Low Overhead Container (LOC) {{LOC}} to define how
audio and video content is packaged. With this packaging mode, each
EncodedAudioChunk or EncodedVideoChunk sample is placed in a separate MOQT
Object. Samples that belong to the same Group of Pictures (GOP) MUST be placed
within the same MOQT Group.

When LOC packaging is used for a track, the catalog packaging attribute
({{packaging}}) MUST be present and it MUST be populated with a value of "loc".

## Time-alignment {#timealignment}
WARP Tracks MAY be time-aligned. Those that are, are subject to the following
requirements:

* Time-aligned tracks MUST be advertised in the catalog as belonging to a common
  render group.
* The render duration of the first media object of each equally numbered MOQT
  Group, after decoding, MUST have overlapping presentation time.

A consequence of this restriction is that a WARP receiver SHOULD be able to
cleanly switch between time-aligned media tracks at group boundaries.

## Content protection and encryption {#contentprotection}

ToDo - content protection for LOC-packaged content.

# Catalog {#catalog}
A Catalog is a MOQT Track that provides information about the other tracks being
produced by a WARP publisher. A Catalog is used by WARP publishers for
advertising their output and for subscribers in consuming that output. The
payload of the Catalog object is opaque to Relays and can be end-to-end
encrypted. The Catalog provides the names and namespaces of the tracks being
produced, along with therelationship between tracks, properties of the tracks
that consumers may use for selection and any relevant initialization data.

The catalog track MUST have a case-sensitive Track Name of "catalog".

A catalog object MAY be independent of other catalog objects or it MAY represent
a delta update of a prior catalog object. The first catalog object published
within a new group MUST be independent.  A catalog object SHOULD only be
published only when the availability of tracks changes.

Each catalog update MUST be mapped to a discreet MOQT Object.


## Catalog Fields

A catalog is a JSON {{JSON}} document, comprised of a series of mandatory and
optional fields. At a minimum, a catalog MUST provide all mandatory fields. A
producer MAY add additional fields to the ones described in this draft. Custom
field names MUST NOT collide with field names described in this draft. The order
of field names within the JSON document is not important.

A parser MUST ignore fields it does not understand.

Table 1 provides an overview of all fields defined by this document.

| Field                   |  Name                  |           Definition      |
|:========================|:=======================|:==========================|
| WARP version            | version                | {{warpversion}}           |
| Delta update            | deltaUpdate            | {{deltaupdate}}           |
| Add tracks              | addTracks              | {{addtracks}}             |
| Remove tracks           | removeTracks           | {{removetracks}}          |
| Clone tracks            | cloneTracks            | {{clonetracks}}           |
| Tracks                  | tracks                 | {{tracks}}                |
| Track namespace         | namespace              | {{tracknamespace}}        |
| Track name              | name                   | {{trackname}}             |
| Packaging               | packaging              | {{packaging}}             |
| Track label             | label                  | {{tracklabel}}            |
| Render group            | renderGroup            | {{rendergroup}}           |
| Alternate group         | altGroup               | {{altgroup}}              |
| Initialization data     | initData               | {{initdata}}              |
| Dependencies            | depends                | {{dependencies}}          |
| Temporal ID             | temporalId             | {{temporalid}}            |
| Spatial ID              | spatialId              | {{spatialid}}             |
| Codec                   | codec                  | {{codec}}                 |
| Mime type               | mimeType               | {{mimetype}}              |
| Framerate               | framerate              | {{framerate}}             |
| Bitrate                 | bitrate                | {{bitrate}}               |
| Width                   | width                  | {{width}}                 |
| Height                  | height                 | {{height}}                |
| Audio sample rate       | samplerate             | {{audiosamplerate}}       |
| Channel configuration   | channelConfig          | {{channelconfiguration}}  |
| Display width           | displayWidth           | {{displaywidth}}          |
| Display height          | displayHeight          | {{displayheight}}         |
| Language                | lang                   | {{language}}              |
| Parent name             | parentName             | {{parentname}}            |

Table 2 defines the allowed locations for these fields within the document

| Location |                Allowed locations for the field                |
|:=========|:==============================================================|
| R        | The Root of the JSON object                                   |
| T        | Track object                                                  |


## WARP version {#warpversion}
Location: R    Required: Yes    JSON Type: Number

Specifies the version of WARP referenced by this catalog. There is no guarantee
that future catalog versions are backwards compatible and field definitions and
interpretation may change between versions. A subscriber MUST NOT attempt to
parse a catalog version which it does not understand.

### Delta update {#deltaupdate}
Location: R    Required: Optional    JSON Type: Boolean

A Boolean that if true indicates that this catalog object represents a delta
(or partial) update. A delta update has a restricted set of fields and special
processing rules - see {{deltaupdates}}. This value SHOULD NOT be added to a
catalog if it is false.

### Add tracks {#addtracks}
Location: R    Required: Optional    JSON Type: Array

Indicates a delta processing instruction to add new tracks. The value of this
field is an Array of track objects {{trackobject}}.

### Remove tracks {#removetracks}
Location: R    Required: Optional    JSON Type: Array

Indicates a delta processing instruction to remove new tracks. The value of this
field is an Array of track objects {{trackobject}}. Each track object MUST include
a Track Name {{trackname}} field, MAY include a Track Namespace {{tracknamespace}}
field and MUST NOT hold any other fields.

### Clone tracks {#clonetracks}
Location: R    Required: Optional    JSON Type: Array

Indicates a delta processing instruction to clone new tracks from previously declared
tracks. The value of this field is an Array of track objects {{trackobject}}. Each
track object MUST include a Parent Name {{parentname}} field.

### Tracks {#tracks}
Location: R    Required: Yes    JSON Type: Array

An array of track objects {{trackobject}}.

### Tracks object {#trackobject}

A track object is JSON Object containing a collection of fields whose location
is specified 'T' in Table 2.

### Track namespace {#tracknamespace}
Location: TFC    Required: Optional    JSON Type: String

The name space under which the track name is defined. See section 2.3 of
{{MoQTransport}}. The track namespace is optional. If it is not declared within
a track, then each track MUST inherit the namespace of the catalog track. A
namespace declared in a track object overwrites any inherited name space.

### Track name {#trackname}
Location: T    Required: Yes   JSON Type: String

A string defining the name of the track. See section 2.3 of {{MoQTransport}}.
Within the catalog, track names MUST be unique per namespace.

### Packaging {#packaging}
Location: T    Required: Yes   JSON Type: String

A string defining the type of payload encapsulation. Allowed values are strings
as defined in Table 3.

Table 3: Allowed packaging values

| Name            |   Value   |      Draft       |
|:================|:==========|:=================|
| LOC             | "loc"     | See RFC XXXX     |

### Track label {#tracklabel}
Location: TF    Required: Optional   JSON Type: String

A string defining a human-readable label for the track. Examples might be
"Overhead camera view" or "Deutscher Kommentar". Note that the {{JSON}} spec
requires UTF-8 support by decoders.

### Render group {#rendergroup}
Location: TF    Required: Optional   JSON Type: Number

An integer specifying a group of tracks which are designed to be rendered
together. Tracks with the same group number SHOULD be rendered simultaneously,
are usually time-aligned and are designed to accompany one another. A common
example would be tying together audio and video tracks.

### Alternate group {#altgroup}
Location: TF    Required: Optional   JSON Type: Number

An integer specifying a group of tracks which are alternate versions of
one-another. Alternate tracks represent the same media content, but differ in
their selection properties. Alternate tracks SHOULD have matching framerate
{{framerate}} and media time sequences. A subscriber typically subscribes to
one track from a set of tracks specifying the same alternate group number. A
common example would be a set video tracks of the same content offered in
alternate bitrates.

### Initialization data {#initdata}
Location: TF    Required: Optional   JSON Type: String

A string holding Base64 {{BASE64}} encoded initialization data for the track.

### Dependencies {#dependencies}
Location: T    Required: Optional   JSON Type: Array

Certain tracks may depend on other tracks for decoding. Dependencies holds an
array of track names {{trackname}} on which the current track is dependent.
Since only the track name is signaled, the namespace of the dependencies is
assumed to match that of the track declaring the dependencies.

### Temporal ID {#temporalid}
Location: T    Required: Optional   JSON Type: Number

A number identifying the temporal layer/sub-layer encoding of the track,
starting with 0 for the base layer, and increasing with higher temporal
fidelity.

### Spatial ID {#spatialid}
Location: T    Required: Optional   JSON Type: Number

A number identifying the spatial layer encoding of the track, starting with 0
for the base layer, and increasing with higher fidelity.

### Codec {#codec}
Location: T    Required: Optional   JSON Type: String

A string defining the codec used to encode the track.
For LOC packaged content, the string codec registrations are defined in Sect 3
and Section 4 of {{WEBCODECS-CODEC-REGISTRY}}.

### Mimetype {#mimetype}
Location: T    Required: Optional   JSON Type: String

A string defining the mime type {{MIME}} of the track.

### Framerate {#framerate}
Location: T    Required: Optional   JSON Type: Number

A number defining the video framerate of the track, expressed as frames per
second.

### Bitrate {#bitrate}
Location: T    Required: Optional   JSON Type: Number

A number defining the bitrate of track, expressed in bits per second.

### Width {#width}
Location: T    Required: Optional   JSON Type: Number

A number expressing the encoded width of the video frames in pixels.

### Height {#height}
Location: T    Required: Optional   JSON Type: Number

A number expressing the encoded height of the video frames in pixels.

### Audio sample rate {#audiosamplerate}
Location: T    Required: Optional   JSON Type: Number

The number of audio frame samples per second. This property SHOULD only
accompany audio codecs.

### Channel configuration {#channelconfiguration}
Location: T    Required: Optional   JSON Type: String

A string specifying the audio channel configuration. This property SHOULD only
accompany audio codecs. A string is used in order to provide the flexibility to
describe complex channel configurations for multi-channel and Next Generation
Audio schemas.


### Display width {#displaywidth}
Location: T    Required: Optional   JSON Type: Number

A number expressing the intended display width of the track content in pixels.

### Display height {#displayheight}
Location: T    Required: Optional   JSON Type: Number

A number expressing the intended display height of the track content in pixels.

### Language {#language}
Location: T    Required: Optional   JSON Type: String

A string defining the dominant language of the track. The string MUST be one of
the standard Tags for Identifying Languages as defined by {{LANG}}.

### Parent name {#parentname}
Location: T    Required: Optional   JSON Type: String

A string defining the parent track name {{trackname}} to be cloned. This field
MUST only be included inside a Clone tracks {{clonetracks}} object.

## Delta updates {#deltaupdates}
A catalog update might contain incremental changes. This is a useful property if
many tracks may be initially declared but then there are small changes to a
subset of tracks. The producer can issue a delta update to describe these changes.
Changes are described incrementally, meaning that a delta update can itself modify
a prior delta update.

A restricted set of operations are allowed with each delta update:
* Add a new track that has not previously been declared.
* Add a new track by cloning a previously declared track.
* Remove a track that has been previously declared.

The following rules are to be followed in constructing and processing delta updates:

* A delta update MUST include the Delta Update {{deltaupdate}} field set to true.
* A delta update catalog MUST contain at least one instance of Add tracks
  {{addtracks}}, Remove tracks {{removetracks}} or Clone Tracks {{clonetracks}}
  fields and MAY contain more. It MUST NOT contain an instance of a Tracks
  {{tracks}} field or a WARP version {{warpversion}} field.
* The Add, Delete and Clone operations are applied sequentially in the order they
  are declared in the document. Each operation in the sequence is applied to the
  target document; the resulting document becomes the target of the next operation.
  Evaluation continues until all operations are successfully applied.
* A Cloned track inherits all the attributes of the track defined by the Parent Name
  {{parentname}}, except the Track Name which MUST be new. Attributes redefined
  in the cloning Object overwrite inherited values.
* The tuple of Track Namespace and Track Name defines a fixed set of Track attributes
  which MUST NOT be modified after being declared. To modify any attribute, a new
  track with a different Namespace|Name tuple is created by Adding or Cloning and then
  the old track is removed.


## Catalog Examples

The following section provides non-normative JSON examples of various catalogs
compliant with this draft.


### Time-aligned Audio/Video Tracks with single quality

This example shows a catalog for a media producer capable of sending LOC
packaged, time-aligned audio and video tracks.

~~~json
{
  "version": 1,
  "tracks": [
    {
      "name": "video",
      "namespace": "conference.example.com/conference123/alice",
      "packaging": "loc",
      "renderGroup": 1,
      "codec":"av01.0.08M.10.0.110.09",
      "width":1920,
      "height":1080,
      "framerate":30,
      "bitrate":1500000
    },
    {
      "name": "audio",
      "namespace": "conference.example.com/conference123/alice",
      "packaging": "loc",
      "renderGroup": 1,
      "codec":"opus",
      "samplerate":48000,
      "channelConfig":"2",
      "bitrate":32000
    }
   ]
}

~~~



### Simulcast video tracks - 3 alternate qualities along with audio

This example shows catalog for a media producer capable of sending 3
time-aligned video tracks for high definition, low definition and medium
definition video qualities, along with an audio track. In this example the
namespace is absent, which infers that each track must inherit the namespace
of the catalog.

~~~json
{
  "version": 1,
  "tracks":[
    {
      "name": "hd",
      "renderGroup": 1,
      "packaging": "loc",
      "codec":"av01",
      "width":1920,
      "height":1080,
      "bitrate":5000000,
      "framerate":30,
      "altGroup":1
    },
    {
      "name": "md",
      "renderGroup": 1,
      "packaging": "loc",
      "codec":"av01",
      "width":720,
      "height":640,
      "bitrate":3000000,
      "framerate":30,
      "altGroup":1
    },
    {
      "name": "sd",
      "renderGroup": 1,
      "packaging": "loc",
      "codec":"av01",
      "width":192,
      "height":144,
      "bitrate":500000,
      "framerate":30,
      "altGroup":1
    },
    {
      "name": "audio",
      "renderGroup": 1,
      "packaging": "loc",
      "codec":"opus",
      "samplerate":48000,
      "channelConfig":"2",
      "bitrate":32000
    }
   ]
}
~~~


### SVC video tracks with 2 spatial and 2 temporal qualities


This example shows catalog for a media producer capable
of sending scalable video codec with 2 spatial and 2 temporal
layers with a dependency relation as shown below:

~~~ascii-figure

                  +----------+
     +----------->|  S1T1    |
     |            | 1080p30  |
     |            +----------+
     |                  ^
     |                  |
+----------+            |
|  S1TO    |            |
| 1080p15  |            |
+----------+      +-----+----+
      ^           |  SOT1    |
      |           | 480p30   |
      |           +----------+
      |               ^
+----------+          |
|  SOTO     |         |
| 480p15    |---------+
+----------+
~~~



The corresponding catalog uses "depends" attribute to
express the track relationships.

~~~json
{
  "version": 1,
  "tracks":[
    {
      "name": "480p15",
      "namespace": "conference.example.com/conference123/alice",
      "renderGroup": 1,
      "packaging": "loc",
      "codec":"av01.0.01M.10.0.110.09",
      "width":640,
      "height":480,
      "bitrate":3000000,
      "framerate":15
    },
    {
      "name": "480p30",
      "namespace": "conference.example.com/conference123/alice",
      "renderGroup": 1,
      "packaging": "loc",
      "codec":"av01.0.04M.10.0.110.09",
      "width":640,
      "height":480,
      "bitrate":3000000,
      "framerate":30,
      "depends": ["480p15"]
    },
    {
      "name": "1080p15",
      "namespace": "conference.example.com/conference123/alice",
      "renderGroup": 1,
      "packaging": "loc",
      "codec":"av01.0.05M.10.0.110.09",
      "width":1920,
      "height":1080,
      "bitrate":3000000,
      "framerate":15,
      "depends":["480p15"]
    },

    {
      "name": "1080p30",
      "namespace": "conference.example.com/conference123/alice",
      "renderGroup": 1,
      "packaging": "loc",
      "codec":"av01.0.08M.10.0.110.09",
      "width":1920,
      "height":1080,
      "bitrate":5000000,
      "framerate":30,
      "depends": ["480p30", "1080p15"]
    },
    {
      "name": "audio",
      "namespace": "conference.example.com/conference123/alice",
      "renderGroup": 1,
      "packaging": "loc",
      "codec":"opus",
      "samplerate":48000,
      "channelConfig":"2",
      "bitrate":32000
    }
   ]
}
~~~

### Delta update  - adding two tracks

This example shows the catalog delta update for a media producer adding
two tracks to an established video conference. One track is newly declared,
the other is cloned from a previous track.

~~~json
{
  "deltaUpdate": true,
  "addTracks": [
      {
        "name": "slides",
        "codec": "av01.0.08M.10.0.110.09",
        "width": 1920,
        "height": 1080,
        "framerate": 15,
        "bitrate": 750000,
        "renderGroup": 1
      }
   ],
   "cloneTracks": [
      {
        "parentName": "video-1080",
        "name": "video-720",
        "width":1280,
        "height":720,
        "bitrate":600000
      }
   ]
}
~~~

### Delta update removing tracks

This example shows a delta update for a media producer removing two tracks
from an established video conference.

~~~json
{
  "deltaUpdate": true,
  "removeTracks": [{"name": "video"},{"name": "slides"}]
}
~~~


### Time-aligned Audio/Video Tracks with custom field values

This example shows catalog for a media producer capable of sending LOC packaged,
time-aligned audio and video tracks along with custom fields in each track
description.

~~~json
{
  "version": 1,
  "tracks": [
    {
      "name": "video",
      "namespace": "conference.example.com/conference123/alice",
      "packaging": "loc",
      "renderGroup": 1,
      "codec":"av01.0.08M.10.0.110.09",
      "width":1920,
      "height":1080,
      "framerate":30,
      "bitrate":1500000,
      "com.example-billing-code": 3201,
      "com.example-tier": "premium",
      "com.example-debug": "h349835bfkjfg82394d945034jsdfn349fns"
    },
    {
      "name": "audio",
      "namespace": "conference.example.com/conference123/alice",
      "packaging": "loc",
      "renderGroup": 1,
      "codec":"opus",
      "samplerate":48000,
      "channelConfig":"2",
      "bitrate":32000
    }
   ]
}

~~~



# Media transmission
The MOQT Groups and MOQT Objects need to be mapped to MOQT Streams. Irrespective
of the {{mediapackaging}} in place, each MOQT Object MUST be mapped to a new
MOQT Stream.

# Timeline track
The timeline track provides data about the previously published groups and their
relationship to wallclock time, media time and associated timed-metadata.
Timeline tracks allow players to seek to precise points behind the live head in
a live broadcast, or for random access in a VOD asset. A timeline track may also
be used to insert events at media times which do not correlate with Object
boundaries. Timeline tracks are optional. Multiple timeline tracks MAY exist
inside a catalog.

## Timeline track payload
The payload of a timeline track is a UTF-8 encoded CSV text file. This payload
is formatted according to RFC4180 "Common Format and MIME Type for
Comma-Separated Values (CSV)" Files {{RFC4180}}. The separator is a comma and
each line is separated by a carriage return. The mime-type of a timeline track
MUST be specified as "text/csv" in the catalog.

Each timeline track begins with a header row of MEDIA_PTS,GROUP_ID,OBJECT_ID,
WALLCLOCK,METADATA. This row defines the 5 columns of data within each record.

* MEDIA_PTS: a media timestamp rounded to the nearest millisecond. This entry
  MUST NOT be empty. If the Object ID entry is present, then this value MUST
  match the media presentation timestamp of the first media sample in the
  referenced Object.
* GROUP_ID: the MOQT Group ID. This entry MAY be empty.
* OBJECT_ID: the MOQT Object ID. This entry MAY be empty.
* WALLCLOCK: the wallclock time at which the media was encoded, expressed as
  the number of milliseconds that have elapsed since January 1, 1970
  (midnight UTC/GMT). For VOD assets, or if the wallclock time is not known,
  the value SHOULD be 0.
* METADATA: a flexible field holding arbitrary string metadata. This field may
  be empty. If not empty, it MUST be enclosed in double quotes. A double-quote
  appearing inside this field MUST be escaped by preceding it with another
  double quote.

## Timeline Catalog requirements
A timeline track MUST carry a 'type' identifier in the Catalog with a value of
"timeline". A timeline track MUST carry a 'dependencies' attribute which
contains an array of all track names to which the timeline track applies.

## Timeline track updating.
The publisher MUST publish a complete timeline in the first MOQT Object of each
MOQT Group. The publisher MAY publish incremental updates in the second and
subsequent Objects within each GROUP. Incremental updates only contain timeline
events since the last timeline Object. Group duration SHOULD NOT exceed 30
seconds.

# Workflow

A WARP publisher MUST publish a catalog track object before publishing any media
track objects.

At the completion of a session, a publisher MUST publish a catalog update that
removes all currently active tracks.  This action SHOULD be interpreted by
receivers to mean that the publish session is complete.


# Security Considerations

ToDo

# IANA Considerations {#IANA}

This document creates a new entry in the "MoQ Streaming Format" Registry
(see {{MoQTransport}} Sect 8).  The type value is 0x001, the name is
"WARP Streaming Format" and the RFC is XXX.

--- back

# Acknowledgments
{:numbered="false"}

- the MoQ Workgroup and mailing lists.
