# Summary

This RFC describes the desire to remove the "Dynamic Audio Buffering" system from libobs. Originally designed to be "automagic" for handling synchronization for OBS sources, this system is highly complex, has far-reaching code impacts, and is not suitable for many broadcast use-cases.

# Motivation

## Software Predictability
A dynamic audio buffer can cause an array of issues in broadcast production. For one case, a user might start an event with a freshly-loaded OBS, and over the course of the stream, a misbehaving source might cause the buffer to dynamically grow up to 940ms. This can be a problem when trying to maintain synchronization between multiple live feeds, as this system allows things to get out of sync without any intervention of the user.

## Code Maintainability
The dynamic audio buffering system requires an extensive amount of compensation code, not just in libobs' audio mixing systems, but in almost every stage of the output pipeline (raw and encoded). Third party outputs have to automatically compensate for buffering changes at any time.

## Existing Alternatives
[This commit](https://github.com/obsproject/obs-studio/commit/e3bdb4ca7bf01aead1ab8b71422dd371bb9425ae) added a fixed audio buffering mode to OBS. With fixed audio buffering, OBS initializes all parts of the audio pipeline to a fixed unchanging value. In my usage of OBS, I always enable fixed buffering mode, with a 200ms buffer size.

# Drawbacks

The ability to set a negative sync offset on a source is governed by the size of the global audio buffer size. For example, if the global audio buffer is set to 200ms, you cannot apply a sync offset of larger than -200ms to a source, without severe audio dropouts and bugs. With the dynamic audio buffering system, this is less problematic as the global audio buffer will grow behind the scenes to accomodate negative sync offsets. However, many users do not understand that this also produces the negative side effects mentioned above. With the removal of dynamic audio buffering, the "Sync Offset" of a source would be restricted by what is considered to be the safe value within the configured audio buffering value.

An alternative workaround for the lack of dynamic audio buffering space would be to add an "audio sync" setting to the render delay filter, allowing audio to be offset into negative values, without adding delays to other sources.

# Additional Information

## UI

The "Low Latency Audio Buffering Mode" setting located in the Advanced section of `Settings -> Audio` would be removed, and replaced with a "Audio Buffering" spinbox, with a minimum value of 21ms, and a maximum value of 940ms. The default value will be 200 or 250ms, and will make a warning appear for values <75ms that audio dropouts may occur more frequently.
