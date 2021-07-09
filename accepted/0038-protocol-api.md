# Summary

- Add an API to register protocol
- Protocols are registered with a list of compatible codecs
- Show only stream encoders compatible with the selected service protocol
- Add as byproduct the possibility to add plugin with new protocol and own output (useless without RFC 39)

# Motivation

Create a better management of outputs, encoders and services with a protocol centered approach.

Byproduct: Add also the possibility to support third-party plugin which add protocol support with his own output.

*Services addition is a part of RFC 39.*

# Design
*This RFC does not consider that because a protocol support it a encoder should be implemented in the main project.*

## Protocols
Add protocol as a new registerable object type in the API. This object will only provide at least his "official" acronym (ex: RTMP, WebRTC) and a list of all supported codecs. Except RTMP, it will only support H.264 and AAC for now.

Protocol types shall be seen as their "specification" put as code, those are present to tell at least which codecs can be used with the protocol maybe even their protocol link prefix (`rtmp://`, `srt://`).
It does not mean that services are compatible with all the codecs present in the specs. And neither mean that OBS support those codecs.

Protocol like SRT are codec agnostic, so adding bool attributes telling that the protocol support any video and audio codecs could be a thing.

Making it separate from outputs and services will make it usable for multi-protocol service from RFC 39 to check if a protocol is registered.

The actually supported protocol with their specification documents :
- RTMP (https://wwwimages2.adobe.com/content/dam/acom/en/devnet/rtmp/pdf/rtmp_specification_1.0.pdf)
- RTMPS (same as RTMP)
- HLS (https://datatracker.ietf.org/doc/html/rfc8216)
- SRT (https://github.com/Haivision/srt/files/2489142/SRT_Protocol_TechnicalOverview_DRAFT_2018-10-17.pdf ; TLDR: codec agnostic)
- FTL (no official one found except this https://github.com/microsoft/ftl-sdk/blob/master/README.md)

## Services
Adding to the API, the required protocol attribute just a char with the protocol name which tell OBS which protocol the service uses.

With RFC 39, OBS could prevent showing services where the protocol is not registered or has no available output compatible with it.
And also make services able to not show unsupported protocol.

We shall consider that if a service handle a protocol, it doens't mean it can handle all the compatible codecs.
So if a service can't handle all the compatible codec, adding a list of supported codec to a service related to his protocol shall be considered.

`rtmp-services` will be modified to obtain this behavior beforehand with only H.264, AAC and Opus as compatible codecs.

## Outputs
Adding to the API, the required¹ protocol attribute just a char with the protocol name which tell OBS with which protocol it can be used.
Also remove the compatible codecs attributes because it is a part of protocol now.
And output should be able to support any codec that the protocol.

A output only support one protocol.

With this, it could be considered to give to the user the possibility to choose between various outputs meant the same protocol as an advanced option.

1. Only for outputs with the flag `OBS_OUTPUT_SERVICE`.

## Encoders
Maybe change codec naming by the "official" ones.

## UI
Changing protocol through service change will update the Output page with the compatible encoders.

Compatible encoders are based on what service can handle and if they specify nothing, any codec accepted by the protocol will be shown.

If the service ask for codec not compatible with the protocol, it will be ignored and written in the log as error.

It also may reset encoder settings if the actual encoder was not compatible.

The user could be able also to choose an audio encoder.

The user may need to be warn about the fact that the encoder was not compatible and he need to setup again his encoder.

When a stream start, OBS will check if the protocol needed by the service have a compatible output if not an error will be emitted.

## Codec naming
For codec naming, we try to get the closiest of an official typography even inside a logo.

- H.264/AVC will be named/changed to `H.264`
- H.265/HEVC will be named `HEVC` since this name is used rather than H.265, only added if adding the fact that a protocol is compatible with it will not cause legal issue.
- AAC we keep `AAC`.
- Opus we keep `opus`.

# Implementation steps

Each step may be in a separated PR

1. Add all API addition to Libobs and plugins which needs it.
2. Add UI changes.

# Drawbacks

- `rtmp-services` will may not work very well with this new behavior. But RFC 39 is here to fix that.

# Benefit
- FTL, if deprecated by OBS and still used by stream services could be separated in a plugin providing the protocol and the output and also service with RFC 39

# Additional Information

You can find my trial to implement this RFC [here](https://github.com/tytan652/obs-studio/tree/protocol_api).