# OBS WebRTC Output


# Summary

OBS should expose APIs to service plugins that allow configuring WebRTC as an output transport.

# Motivation

WebRTC is a popular method for transmitting live video. Services such as Millcast, Wowza, Janus, and Caffeine have added WebRTC ingest support. In order to support these services there are at least two forks of OBS. This provides a worse experience for the end user and wastes development work maintaining these forks.

Exposing the API prevents service plugins from depending on behavior in the underlying WebRTC library implementations. This allows plugins to be updated less often and prevents them from breaking between OBS releases. It also reduces final plugin size.

WebRTC does not specify how clients should exchange SDP messages, leaving it up to each service. Each service plugin will be responsible for implementing a version of this for how their service works. To aid this I suggest shipping more APIs that let them use OBS managed Websockets and http calls.

The [WebRTC fork](https://github.com/CoSMoSoftware/OBS-studio-webrtc) of OBS has code quality issues. The output plugins it provides are directly coupled to libwebrtc. They also are not isolated from each other and code for multiple services is shared in the same file. This makes shipping them separately or from a plugin manager impossible.

# Implementation Details

## WebRTC library

OBS already has ways to capture and encode audio and video making a library like libwebrtc very bloated for our uses. I propose we use the following library: 

**[amazon-kinesis-video-streams-webrtc-sdk-c](https://github.com/awslabs/amazon-kinesis-video-streams-webrtc-sdk-c)**

* Written in C so it can integrate nicely with libobs.
* Supports mbedTLS which OBS already uses so we have one less dependency to ship.
* Very small, sub 200k library size. I built a test app with the library and its dependencies statically linked and it was around 4 MB.

## Signaling

Services should have multiple choices for implementing the signaling layer.

### Websockets
The most common signaling protocol. We should provide Wrapper functions around libwebsocket to help plugins safely send and receive messages.

### HTTPS
OBS has a curl dependency, we can either expose this directly or provide wrapper functions.

### Whip
Whip greatly simplifies the signaling layer, it only requires a POST request with an SDP and an Authorization header with a bearer token.

<https://datatracker.ietf.org/doc/draft-murillo-whip/>
<https://millicast.medium.com/whip-the-magic-bullet-for-webrtc-media-ingest-57c2b98fb285>

Usage examples:
* Use WHIP with just the services file and no code. 
* Provide an easy way for services to do oauth that we can use with WHIP.
* Let services call WHIP with their own auth flow / code

### Scripting?
We should investigate the possibility of allowing services to implement their signaling layer with the current OBS scripting system. Allowing services to ship a single Python or Lua file might be preferable than only allowing C/C++ shared library based plugins.

## New Public APIs

```
/**
	Pass the current encoder settings to the rtp tranceiver settings in the webrtc library. This will make sure the local SDP has the correct codecs.
*/
obs_webrtc_configure_tranceiver(obs_service_t *service, const obs_encoder_t *encoder);

/**
	Simple enum to say if an SDP is an offer or an answer
*/
enum sdp_type {offer, answer};

/**
	Set the remote SDP for a webrtc connection. This is a value that the remote server will give you during signaling.
*/
obs_webrtc_set_remote_description(obs_service_t *service, sdp_type type, const char *sdp);

/**
	If the remote server has sent us an offer remote description this can be called to generate the SDP to respond with.
*/
char* sdp = obs_webrtc_create_answer(obs_service_t *service);

/**
	Set the SDP we use locally, generally returned from `obs_webrtc_create_answer`
*/
obs_webrtc_set_local_description(obs_service_t *service, sdp_type type, const char *sdp);

/**
	With whip we will need to generate an offer first
*/
char* sdp = obs_webrtc_create_offer(obs_service_t *service);
```

# Concerns / Questions
* Should we respect picture loss packet (PLI) or should OBS send keyframes as it already does?
* Are there any issues being SDP based? ex. media soup requires an adapter to use SDP <https://github.com/versatica/mediasoup-sdp-bridge>

# Q&A

## What does this mean for me as a plugin author?
Your plugin will not link against the chosen WebRTC library directly. OBS will provide functions to get an SDP to send to a remote service and also a call to consume an SDP from the remote service. We will also ship APIs to allow your plugin to communicate over a Websocket or make HTTP calls without having to link against curl or a Websocket library.

# Additional Information / Notes

## Open source webrtc libraries
* **libwebrtc** - The library that everyone uses moves fast and a large codebase but does everything.
* **[webrtc-rs](https://github.com/webrtc-rs/webrtc)** - A rust implementation of webrtc based on pion.
* **[pion](https://github.com/pion/webrtc)** - Golang webrtc library very modular
* **[amazon-kinesis-video-streams-webrtc-sdk-c](https://github.com/awslabs/amazon-kinesis-video-streams-webrtc-sdk-c)** - C based webrtc library

## Open source Websocket libraries

* **[libwebsockets](https://libwebsockets.org)** - Pure c web socket library
* **[WebSocket++](https://www.zaphoyd.com/projects/websocketpp/)** - Header only C++ library that implements RFC6455 (The WebSocket Protocol) 
* **[Boost Beast](https://www.boost.org/doc/libs/1_76_0/libs/beast/doc/html/beast/using_websocket.html)** - Beast is a C++ header-only library serving as a foundation for writing interoperable networking libraries by providing low-level HTTP/1, WebSocket, and networking protocol vocabulary types and algorithms using the consistent asynchronous model of Boost.Asio.


