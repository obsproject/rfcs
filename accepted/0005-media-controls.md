- Start Date: 2020-01-13
- RFC PR: #6
- Mantis Issue: https://obsproject.com/mantis/view.php?id=329

- Related PRs:
	1. https://github.com/obsproject/obs-studio/pull/2300
	2. https://github.com/obsproject/obs-studio/pull/2274
	3. https://github.com/obsproject/obs-studio/pull/1803

# Summary

Create the ability for users to control the media and VLC sources.

# Motivation

Users should be able to have more control over media sources. With a media control widget, they can see how much time is left in the video, seek to a certain point, play/pause, etc.

# Detailed design

## User UX

- Have the media controls as a single widget under or near the preview.
- Slider: The slider would be used to seek to specific points in the media
- Timer labels: on the sides of the slider (show the length of the media and show the current time of it)
- Buttons: Previous, Play/Pause/Restart, Stop, Next

- Config button: A dropdown would show up showing several options
	1. Properties (open properties of current source)
	2. Filters (open filters of current source)
	3. Speed (0.25x, 0.5x, 0.75x, 1.0x, etc.), set speed of current source
	4. Loop (sets whether to loop the media)

## Other conderations

- The media source currently doesn't have playlist, play/pause, seek support (PRs mentioned above implement these)
- VLC source needs seek support (https://github.com/obsproject/obs-studio/pull/1803 adds this)
- Ideally the media source should have feature parity with the VLC source

# How We Teach This

Have the widget on by default, so users don't have to dig through settings or menus to able it.

# Drawbacks

More UI would take up more space.

# Alternatives

The VLC source currently has hotkeys to control media. Hotkeys are not ideal because users may not know that they exist.
