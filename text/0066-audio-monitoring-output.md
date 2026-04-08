- Start Date: 2026-04-08
- RFC PR: #66

# Summary

Move audio monitoring out of libobs and make it an output plugin.

# Motivation

Currently, when a source is being monitored, a new audio monitor is created for each source. The goal here is to have
a single mix where all of the sources are output to, thus avoiding sync issues when having multiple sources being
monitored.

# Detailed design

- This would be an raw audio output
- Each output can have its own mix, this is set by obs_output_set_mixers()
- The mix would be hidden from the UI and other outputs
- When a user selects a source to be monitored, the source mixer is set by obs_source_set_mixers()

# Other conderations

- In the future we could have multiple audio monitoring busses, so creating a new bus would be as simple as creating a
new output and setting a mixer for it.

# How We Teach This

Nothing would change in the UI.

# Alternatives

Changing libobs to have audio tracks, like we currently have with video tracks. This is kinda redudant, since we have
audio mixes already.
