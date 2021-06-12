# OBS WebRTC Output


# Summary

OBS should expose APIs to service plugins that allow configuring WebRTC as an output transport.

# Motivation

WebRTC is a popular method for transmitting live video. Services such as Millcast, Wowza, Janus, and Caffeine have added WebRTC ingest support. In order to support these services there are at least two forks of OBS. This provides a worse experience for the end user and wastes development work maintaining these forks.

Exposing the API prevents service plugins from depending on behavior in the underlying WebRTC library implementations. This allows plugins to be updated less often and prevents them from breaking between OBS releases. It also reduces final plugin size.

WebRTC does not specify how clients should exchange SDP messages, leaving it up to each service. Each service plugin will be responsible for implementing a version of this for how their service works. To aid this I suggest shipping more APIs that let them use OBS managed Websockets and http calls.

The [WebRTC fork](https://github.com/CoSMoSoftware/OBS-studio-webrtc) of OBS has code quality issues. The output plugins it provides are directly coupled to libwebrtc. They also are not isolated from each other and code for multiple services is shared in the same file. This makes shipping them separately or from a plugin manager impossible.

# Drawbacks

If using libwebrtc we have to keep up with the release cadence or breaking changes.

There is now more code to test / maintain if we don't manage to onboard devs from the services.

# Additional Information
## What webrtc library?
* **libwebrtc** - The library that everyone uses moves fast and a large codebase but does everything.
* **[webrtc-rs](https://github.com/webrtc-rs/webrtc)** - A rust implementation of webrtc based on pion.
* **[pion](https://github.com/pion/webrtc)** - Golang webrtc library very modular
* **[amazon-kinesis-video-streams-webrtc-sdk-c](https://github.com/awslabs/amazon-kinesis-video-streams-webrtc-sdk-c)** - C based webrtc library

## What Websocket library?

?

## What does this mean for me as a plugin author?
 Your plugin will not call libwebrtc directly. OBS will provide functions to get an SDP to send to a remote service and also a call to consume an SDP from the remote service. We might also ship APIs to allow your plugin to communicate over a Websocket or make HTTP calls without having to link against curl or a Websocket library.

