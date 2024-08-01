# Consistent behavior for video sources that do not have video

## Summary / Abstract
To fix the problem of what to do when a video source has no video, we create a system for the user to indicate what they want the source to do.
Presented to the user as a separate setting, the user can choose between invisible, the last frame, or a black image the size of the last frame.
OBS handles when a source indicates that it no longer has video.


## Motivation
Some video sources, like window capture, video capture, or media sources, at some point in their lifetime might not have any video to display.
Currently, what sources do is that scenario is pretty much a wild west.
For example, *macOS Screen Capture* shows the last known frame, *Media Sources* have a checkbox between invisible and the last known frame, *Game Capture* (Windows) shows nothing (to my knowledge), etc.
Past discussions have shown that different users expect different things to happen in this case.
Some want the last frame, others want nothing, etc.
Part of this are privacy concerns on what is shown, however this goes both ways: A window capture that shows the last frame may not be supposed to do that as the user clicked the window away since they showed something they didn't mean to; meanwhile a source that suddenly shows nothing could expose what is below it.
Giving control over this to the user should address those conerns.

## Details
### New APIs
The problem with the current situation is that every source needs to implement its own property to show basically the same thing.
To simplify this, the idea is to not use the source properties/settings API.
We create a setting that's not stored as part of the usual source settings.

A video source that at some point might not have video it should indicate so via a new capability flag (The exact value of this might change if `(1 << 17)` is taken by the time of implementation) in its `obs_source_info`:
```c
#define OBS_SOURCE_CAP_MIGHT_NOT_HAVE_VIDEO (1 << 17)
```
Setting this flag requires that the source implements and respects the other API's shown below.
Any source that might not have video should set this.
The idea of a new flag is that not all sources (e.g., text sources or color sources) require such a setting, so they should not get a UI for this or need to implement new APIs.


We create an enum to for the available states, as well as methods to get this from and set it on the source.
```c
enum obs_source_no_video_behavior {
    OBS_SOURCE_NO_VIDEO_BEHAVIOR_SHOW_BLACK_FRAME,
    OBS_SOURCE_NO_VIDEO_BEHAVIOR_SHOW_NOTHING,
    OBS_SOURCE_NO_VIDEO_BEHAVIOR_SHOW_LAST_FRAME,
}
enum obs_source_no_video_behavior obs_source_get_no_video_behavior(obs_source_t *source);
void obs_source_set_no_video_behavior(obs_source_t *source, enum obs_source_no_video_behavior behavior);
```

`obs_source_info` gets extended with a `has_video` callback (see "Synchronous sources" for context):
```c
struct obs_source_info {
/* ... */
    bool (*has_video)(void *data);
};
```

### Source Behavior
#### Async sources
If an async source at some point no longer has new video to output (e.g., because a Video Capture Device gets disconnected), it should call `obs_source_output_video(source, NULL)`.
Note that passing `NULL` as the parameter no longer means that no video will be shown - libobs will handle the actual video output.

Asynchronous sources should not call `obs_source_output_video(source, NULL)` if they do not set the `OBS_SOURCE_CAP_MIGHT_NOT_HAVE_VIDEO` flag.
Doing so results in undefined behavior.

#### Synchronous sources
Synchronous sources that have the `OBS_SOURCE_CAP_MIGHT_NOT_HAVE_VIDEO` flag set must implement the `has_video` callback.
This will get called before rendering the source.
If the source does not have video, it should return `false` in `has_video`.
In this case, `video_render` will not get called; instead libobs will handle the video that gets shown.

### UI
A final UI for this should be decided by dedicated UI/UX discussions, but here are suggestions:

Ideally, this is part of a tabbed properties system ([example](https://github.com/obsproject/obs-studio/pull/8155)).
If a source has set the `OBS_SOURCE_CAP_MIGHT_NOT_HAVE_VIDEO` flag, it would show one combobox or radio button with the three options:
```
When no video is shown:
- Show a black image
- Show nothing
- Show the last frame
```

If tabbed properties are not an option, the options could be radio button added at the bottom of the source's properties.
However, it would not technically be a property and not set any "normal" source settings (though to the user it would look like a normal property).

Showing the black image is the default behavior, and is also what gets returned by `obs_source_get_no_video_behavior` by default.

The setting selected by the user gets set on the source in libobs via `obs_source_set_no_video_behavior`.

### Migration
As mentioned above, by default a black frame should get shown for new sources.
Also doing so for existing sources might be a breaking change for some setups, but makes sense for consistency.

Some sources however already have similar source properties/settings.
This includes:
- *Media Source*: Existing sources that have "Show nothing if playback ends" enabled should get migrated to `SHOW_NOTHING`, others to `LAST_FRAME`. The setting should be removed.
- *Image Slide Show*: Existing sources that have "Hide slideshow when done" enabled should get migrated to `SHOW_NOTHING`, others to `LAST_FRAME`. The setting should be removed.
- *Other?*: I cannot think of other sources, but if there are any, please comment so!

Migrations should be done by the UI.
