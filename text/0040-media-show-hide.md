# Summary

Provide more nuanced control for when media sources start playing

# Motivation

FFMpeg media sources currently either autoplay as soon as the OBS session is started, or (if the media source is configured with "Restart playback when source becomes active") will start/restart when the scene is activated.

VLC media sources have slightly more nuance, with visibility behaviors for (re)start/stop, play/pause, or always play. 

This RFC proposes to expand on the nuance from the VLC media source to:
- provide the user better control of the media behaviors
- provide consistency between the media playing sources

This nuance supports a few unmet use cases, including:
- fully manual playback of media sources 
- hybrid controls (automatic starting/manual stop, or manual start / automatic pause)
- "pause when not visible/unpause when visible"-parity for FFMpeg sources.

# Proposed changes

In order to streamline the options available, this RFC proposes splitting the current "Visibility behavior" option into two separate controls for activate ("show") and deactivate ("hide") events. In order to maintain backwards compatibility, an additional setting ("Autoplay") is needed to support playing the media immediately upon starting the OBS session.

![UI Sketch](https://user-images.githubusercontent.com/111218/120713020-38057980-c476-11eb-97a2-8763ce9efeb6.png)

(For brevity, this sketch above omits non-playback media source options; also, no visual design is implied above)

The proposed properties should be equally applicable to both the FFMpeg and VLC media sources (and possibly the slide show?).

## Proposed technical implementation

Beyond adding the properties and updating the source callbacks for activate/deactivate and create/update, it may be useful to add an additional `OBS_MEDIA_STATE` for "loaded and ready to play" to distinguish it from media that is playing/has already been played.

# Drawbacks

This change increases the complexity of the options available to the user. Some combinations of options might not make sense ("Play" and "Restart" on show are equivalent if "Close file" on hide is selected) 

The "autoplay" checkbox is slightly unfortunate label, and may need additional nuance or hints to make it clear that it controls the start-up behavior and is unrelated to autoplaying when the scene is activated.

# Related content

- https://github.com/obsproject/obs-studio/issues/4778
- https://github.com/obsproject/rfcs/blob/master/accepted/0005-media-controls.md

