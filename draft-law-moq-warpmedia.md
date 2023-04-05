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
 - next generation
 - unicorn
 - sparkling distributed ledger
venue:
  group: "Media Over QUIC"
  type: "Working Group"
  mail: "moq@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/moq/"
  github: "wilaw/MoQ"
  latest: "https://wilaw.github.io/MoQ/draft-law-moq-warpstreamingformat.html"

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

informative:


--- abstract

This document specifies the WARP Media Format, designed to operate on MoQ Transport.


--- middle

# Introduction

WARP Media Format (WMF) is a media format designed to deliver CMAF compliant media content over the MoQ transport. WMF leverages a simple priotization strategy of assigning newer content a higher send priority, allowing intermediaries to drop older data, and video over adio, in the face of congestion. 


# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Catalog format

# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.

# Contributors
*Alan Frindell
*Ali Begen
*Charles Krasic
*Cullen Jennings
*Hang Shi
*James Hurley
*Jordi Cenzano
*Mike English


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
