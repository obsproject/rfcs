# Summary

Fragmented MP4, or fMP4, addresses the drawback of MP4/MOV files requiring "finalisation" by splitting the file into "fragments" that can be read and decoded independently.
This means that if a file is not finalised it is still readable up until the penultimate fragment.

Functionally this can already be manually done by setting the following custom muxer options: `movflags=frag_keyframe+empty_moov+delay_moov`.  
This will fragment the file on keyframes, essentially making the file recoverable up until the last GOP should the file not be finalised (e.g. FFmpeg muxer process crash or unexpected system shutdown).

Despite this being generally referred to as "fragmented mp4", the same can be applied to MOV, which is a sister format to MP4.

This RFC proposes making this the default for the MP4/MOV container family.  
We may also consider changing the default container from MKV to MP4/MOV if real-world tests show it to be reliable enough.

# Motivation

While the current default format MKV is resilient against crashes, it does not have the greatest compatibility.
Most browsers, some video players, and especially many video editors do not support MKV containers (properly).

There are also some known issues with MKV related to how it stores the video's frame rate that can result in issues with playback/seeking in editors,
as well as potential issues when writing files with a very high bitrate (such as ProRes) on an I/O limited machine (e.g. HDD or network share).

Additionally, with the upcoming ProRes support generally being expected to use MOV, having a higher resilience for that container as well is desirable.

Another benefit is that platforms such as YouTube can start transcoding while the file is still being uploaded, akin to a "faststart" mp4.

# Implementation

The proposed implementation would show a new checkbox for enabling fragmented recording when MP4 or MOV is selected (default: on).
This checkbox will have a tooltip explaining fragmented recording and includes a notice about remuxing potentially still being required for compatibility with some older software

If the checkbox is enabled the muxer option `movflags` will be set to `frag_keyframe+empty_moov+delay_moov`, unless the user has already manually specified `movflags`.

Additionally the current MP4/MOV warning would only be shown if fragmentation is disabled.

# Drawbacks

While fragmented MP4 is well supported in most video players, browsers, and editors, some older or more niche software may not fully support it.

Viewing the raw file via a HTTP URL may be slow in some browser (e.g. Chrome), as they require reading all fragment metadata before playback.
This is especially true if the CDN/Origin do not support range requests.

Additionally, the fragmented nature of fMP4 can result in a small amount of overhead when compared to a traditional MP4 file.
Although in many cases the file will actually be slightly smaller.

# Additional Information

A general introduction to the MP4 format, also converting fMP4, can be found here: https://www.agama.tv/demystifying-the-mp4-container-format/
