# Summary

Add the ability to render multiple "video mixes" where a "video mix" is ultimately a `render_texture` created by compositing scenes and sources that can be fed into one or more video encoders/outputs. 


# Motivation

OBS supports multiple video outputs allowing simultaneous streaming, recording, virtual camera, etc. These outputs, however, must all be derived from a single `render_texture`. This makes it impossible to support [Multiple Video Outputs (Selective Recording, ISO recording)](https://ideas.obsproject.com/posts/41/multiple-video-outputs-selective-recording-iso-recording) or to share just a webcam via virtual camera (to participate in a video conference while streaming) with the current architecture.

While plugins do exist to implement these features, they do so in a roundabout way. They provide a no-op filter which intercepts video from a source for the plugin to process. This completely bypasses the normal output pipeline of OBS.


# Design

## Split `obs_core_video`

Video rendering data is stored in a single instance of the `obs_core_video` struct internally. This struct contains shared resources like the `graphics_t` pointer and `gs_effect_t`s and state management for the main rendering thread, but it also contains data specific to the render such as `render_texture` and all its related copy/staging surfaces, gpu encoder thread management, and the `video_t` pointer. The first step will be to factor out those `render_texture` related fields into a new `obs_core_video_mix` struct.

The initial implementation will bias toward leaving fields in `obs_core_video` whenever possible to limit the scope of the refactor. Settings like fps and resolution will be the same for all video mixes. Future work can be done to make these configurable per video mix, but that functionality is out of scope for the initial refactor.

## Multiple Renders per View

The `obs_view` struct already supports up to 64 `source_t` channels of input to the render thread. All rendering of video, however, is done to the same texture (one on top of the other). Currently only 7 of those channels are used and only 1 for video (channel 0 is the primary video source and 1-6 are audio device sources).

Adding more video sources to the existing `main_view` is an obvious way to support multiple video mixes. In order to get useable output textures, the render pipeline will be updated to use a different `obs_core_video_mix` struct and therefore a separate `render_texture` and related outputs for each channel with a video source set. Theres no real need to continue to support multiple sources rendering onto the same texture within a view since the same thing can (and is) implemented with scene or transition sources today.

The `obs_core_video` struct will be updated to contain `MAX_CHANNELS` copies of the new `obs_core_video_mix`, one for each potential channel in the `main_view`. The main render thread will loop through each channel and render any source with video to its own dedicated `render_texture`. Each channel may also have separate encoders/outputs and a dedicated gpu thread as needed.

## Resource Management

Additional video mixes each require their own dedicated graphics resources. This proposed design supports up to 64 mixes, but pre-allocating graphics resources when most channels will go unused would be wasteful. Instead, initialization for `render_texture` and related resources will occur during the first render loop where a channel has a `source_t` with video assigned.

It may be possible to free resources early as well if the `source_t` is removed, but this introduces risks since those resources are shared with the gpu encoder and output threads and coordinating tear down might be challenging. It's not clear that early cleanup would provide much real-world benefit, so this optimization will likely be omitted from the initial implementation.


# Proposed UX

While this proposal focuses primarily on the `libobs` refactor required to enable multiple video mix functionality, here are some ideas on how the UI could be updated to take advantage:

## Simple Output Mode

An additional "Virtual Camera" `GroupBox` could be added to the Output settings when in Simple mode to select the camera source. It would default to "Same as stream" but could be overridden to select any other source. If an override was configured, this source could be set to a static channel >6 which the virtual camera could use when starting.

Source selection could be limited to only Video Capture sources in Simple mode in order to streamline the configuration for the most common use-cases.

Configuring ISO recording is likely beyond the scope of what makes sense to include in the Simple configuration mode.

## Advanced Output Mode

A new "Video" tab could be added to the Output settings when in Advanced mode. This would function similarly to the existing "Audio" tab where users could configure scenes/sources for each of N video tracks. Additionally, the virtual camera source selection would be shown here while in Advanced mode. This part of the configuration would simply map sources to channels very much like how the Advanced Audio Properties dialog enables mapping audio sources to different tracks. The only difference being that video tracks don't mix multiple sources directly, a scene must be used to setup this mixing separately if desired.

The Recording tab could also be updated to configure ISO recording by selecting which video tracks to record (similar to selecting audio tracks to record except that each video would be written to a separate file). A basic version of this might simply allow selecting which video tracks to record and then duplicating the configured encoder under the hood to record any selected track with the same settings. A more advanced version might allow full configuration for each track with different encoder settings, separate audio track selections per video, etc. Either of these or other options can easily be implemented on top of the core `libobs` changes proposed above.


# Alternatives

## Use Multiple `obs_view`s

Instead of re-purposing the channels within the `main_view`, it would be possible to add similar multi-rendering support using separate `obs_view` objects for each. This ultimately would be a bigger refactor to the rendering thread and how sources are added/removed for rendering without any clear benefit vs the proposed approach.

## Use Source Filters

There are several existing plugins which use source filters to intercept texture data and implement similar functionality. This bypasses most of the core render pipeline and would make it much more difficult to manage gpu encoder count limitations, consistent frame dropping, configuration, etc.


# Drawbacks

Deferring resource allocation could hide graphics failures that would have otherwise resulted in a fallback from DirectX to OpenGL. Not all resource creation is deferred, so this would only happen if some resources could be created but not all of them. If this ends up being an actual issue then the allocation for channel 0 resources (the default video channel) can be hoisted back into the main init call and treated specially relatively easily.

Re-purposing the `obs_view` channels to render to separate textures removes the ability to perform the sort of layered rendering to a single texture that exists today. It does not seem like this functionality is actually being used, nor what use-case might require it that couldn't be similarly accomplished with a scene instead.

Using a fixed-size channel list limits the available video mixes. Currently 64 channels are support (with 7 already in use), but realistically a PC is unlikely to be able to handle 58 separate video mixes with individual encoders and outputs all at once anyway. If there's a need to support more then future work could either make the channel count dynamic or simply increase the `MAX_CHANNELS` constant at the cost of a minimal amount of additional memory consumption.

# Additional Information

Plugins offering similar functionality:
* [Source Record](https://obsproject.com/forum/resources/source-record.1285/)
* [OBS Virtualcam](https://obsproject.com/forum/resources/obs-virtualcam.949/) and [Virtual Cam Filter](https://obsproject.com/forum/resources/virtual-cam-filter.1142/)

Implementation:
* I have a locally working version of the proposed changes, including a hacky PoC of using a non-default source for the virtual camera. I'll add a link here once I open up a PR.
