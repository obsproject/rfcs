- Start Date: 2020-01-17
- RFC PR: #9
- Mantis Issue: none

- Related PRs:
	1. https://github.com/obsproject/obs-studio/pull/1045

# Summary

Add the ability for users to output audio monitoring to multiple devices.

# Motivation

Users want to be able to output to multiple audio devices. For example they could monitor the audio with headphones and also output the audio to speakers at the same time. They also don't have to use other software for this.

# Detailed design

## User UX

- In https://github.com/obsproject/obs-studio/pull/1253, a headphone checkbox is added to the volume control widget (needs to be merged).
- In a right click menu for the headphone checkbox, add items for Bus A and Bus B where users would click to enable the audio busses.
- Selection of busses, would be in the settings dialog where the audio monitoring currently is.

## Other

- Lately I've been greatly simplifying and refactoring the code of the previous PR above.
- My new code would allow for unlimited audio busses in the backend. It currently has 2 output busses.
- The new code: https://github.com/cg2121/obs-studio/tree/audio-busses

# Drawbacks

The selection of the busses would be in a context menu.

# Alternatives

Use other software for audio routing.
