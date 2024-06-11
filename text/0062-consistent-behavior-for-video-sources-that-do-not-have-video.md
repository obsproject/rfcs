# Consistent behavior for video sources that do not have video

## Summary / Abstract
To fix the problem of what to do when a video source has no video, we create a system for the user to indicate what they want the source to do.
Presented to the user as a separate setting, the user can choose between invisible, the last frame, or a black image the size of the last frame.
The source queries OBS once it no longer has video to find the status of that setting and act accordingly.


## Motivation
Some video sources, like window capture, video capture, or media sources, at some point in their lifetime might not have any video to display.
Currently, what sources do is that scenario is pretty much a wild west.
For example, *macOS Screen Capture* shows the last known frame, *Media Sources* have a checkbox between invisible and the last known frame, *Game Capture* (Windows) shows nothing (to my knowledge), etc.
Past discussions have shown that different users expect differnt things to happen in this case.
Some want the last frame, others want nothing, etc.
Part of this are privacy concerns on what is shown, however this goes both ways: A window capture that shows the last frame may not be supposed to do that as the user clicked the window away since they showed something they didn't mean to; meanwhile a source that suddenly shows nothing could expose what is below it.
Giving control over this to the user should address those conerns.

## Details
### New APIs / Internals
The problem with the current situation is that every source needs to implement its own property to show basically the same thing.
To simplify this, the idea is to not use the source properties/settings API.
We create a setting that's not stored as part of the usual source settings and can be queried by the source.

A video source that at some point might not have video it should indicate so via a new capability flag (The exact value of this might change if (1 << 17) is taken by the time of implementation) in its `obs_source_info`:
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
If a source at some point has no video, it should query what state is set via
`obs_source_get_no_video_behavior`
According to what is set, it is then the source's responsibility to output what is asked of it.
For this, sources should remember the last frame and its dimensions.
The value of this can change at any time (for the avoidance of doubt, this includes when there currently is no video), and should always be respected.
If a source is supposed to show the last frame but never had one, it should show a black frame instead.
The same applies if it receives an unknown value (e.g., when a future version of OBS adds new values but the source was not yet updated).

### UI
A final UI for this should be decided by dedicated UI/UX discussions, but here are suggestions:
Ideally as a part of a tabbed properties system ([example](https://github.com/obsproject/obs-studio/pull/8155)), the UI for this would be part of a separate tab.

Here, if a source has set the `OBS_SOURCE_CAP_MIGHT_NOT_HAVE_VIDEO` flag, it would show one combobox or radio button with the three options:
```
When no video is shown:
- Show a black image
- Show nothing
- Show the last frame
```
Showing the black image is the default behavior, and is also what gets returned by `obs_source_get_no_video_behavior` by default.

The setting select here by the user gets set on the source in libobs via `obs_source_set_no_video_behavior`.

### Migration
As mentioned above, by default a black frame should get shown for new sources.
Also doing so for existing sources might be a breaking change for some setups, but makes sense for consistency.

Some sources however already have similar source properties/settings.
This includes:
- *Media Source*: Existing sources that have "Show nothing if playback ends" enabled should get migrated to `SHOW_NOTHING`, others to `LAST_FRAME`. The setting should be removed.
- *Image Slide Show*: Existing sources that have "Hide slideshow when done" enabled should get migrated to `SHOW_NOTHING`, others to `LAST_FRAME`. The setting should be removed.
- *Other?*: I cannot think of other sources, but if there are any, please comment so!

Migrations should be done by the UI.

### Implementation
Of course, the libobs changes and the UI changes need to be done for any source to implement this.
It would be helpful if all built-in sources could implement this at the same point, so that it's consistent across the program, but doing so can be optional - it might be necessary to do this is steps due to the variety of source types, especially across operating systems.

## Drawbacks
Sources still need to implement all of this themselves, and especially third-party sources might not do so.
An alternative would be having libobs handle more of this (keeping the last frame, etc).
However, this would still require sources to indicate at which point they have no video, and overall seems more complicated to me.

## Additional Information
There is no code written for this yet.
I'm happy about any and all thoughts on and suggestions for this!
Both about what people think about the concept in general (as mentioned, there could be other ways to solve this), and whether they have better ideas for names of the new APIs.
