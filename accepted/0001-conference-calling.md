- Start Date: 2019-11-15
- RFC PR: #2
- Mantis Issue: N/A

# Summary

Implement an over the web conference calling system that can be joined through a web URL, with separate video and audio feeds for each participant in OBS.

# Motivation

Currently the only option for easily bringing in separate video and audio feeds of a conference call into OBS is with Skype outputting NDI, and being picked up by the obs-ndi plugin. This solution requires the user to install a plugin that doesn't ship with OBS, and to use Skype which puts a watermark over the NDI feed. The solution is also almost entirely closed source.

Having an open source and peer to peer system that can directly ingest into OBS would allow for much easier collaboration and remote workflows in OBS.

# Detailed design

The calling should be done over WebRTC, so that anyone with a modern web browser will be able to simply receive a link in order to join a call. As WebRTC is a big binary on it's own, the system in OBS should utilize a compatible but lightweight library. Since we'll want to be dealing with data directly, this can be achieved with either [librtcdc](https://github.com/xhs/librtcdc) or [librtcdcpp](https://github.com/chadnickbok/librtcdcpp), which implement only the data channel parts of the WebRTC standard.

In Tools, there should be a menu item for creating calls. On clicking the menu item, a dialog should come up with an obsproject.com URL that can be copied to people who will participate in the call, an option to pick a webcam and mic to transmit back to the participants, and an option to mute the call mix going out to desktop audio. The OBS client will then communicate with a server running on obsproject.com to wait for clients to peer with it using WebRTC.

When a participant opens the URL and grants permissions, it should communicate with the OBS client its capabilities (video, audio, screenshare, etc), and wait for the OBS client to request a media stream. When the client is requested to provide a stream, the webpage should start a MediaRecorder() instance and send the raw byte data over the WebRTC data channel.

OBS will receive the video and audio over the WebRTC data channel, and send it to libavcodec to decode. It will then provide a video feed back over the data channel to all users that switches between cameras based on who is speaking. This will contain an audio mix of all participants.

OBS should add a source for displaying one of these decoded feeds. This should contain a drop down to pick the source to pull into OBS, containing the following options.
- Active Speaker, that will contain a mix of everyones audio, and will automatically switch video based on who is talking.
- Active Remote Speaker, that does the same as above, but will not show the local feed.
- Separate feeds for each of the participants in the call.

The server hosted by OBS Project should also be open source. It will not handle broadcasting any of the video itself, merely facilitate the handshake between peers.

# How We Teach This

We will definitely need a guide written about using this system. A video will have to be made if this releases as part of an update. The source for call participants should have a label pointing people to the menu in tools to create a call, as a means of discoverability.

# Drawbacks
This requires OBS Project to host a server to facilitate P2P handshakes. Calling will be down if the host for the site is ever down, since a URL needs to be generated. Serving static HTML that can be cached for an entire directory may be a solution for this, if the server doesn't have to be active to handle the peering.

# Alternatives
Compiling OBS with the full WebRTC library and using the video channel instead. Another option is to provide a solution based around the existing browser source (although this may result in several more WebRTC connections to display feeds). Alternatively, we could find an existing open source conferencing program with a compatible license that could be integrated into OBS directly.