# Summary

Add HLS ingestion as one of the live streaming options in OBS.

# Motivation

[HLS](https://tools.ietf.org/html/draft-pantos-hls-rfc8216bis-07) live streaming
is supported by a number of platforms. HLS can support next generation codecs
that RTMP does not. Newer codecs can offer much better compression relative to
H.264, allowing users to stream with higher quality for a given bitrate, or
stream with the same quality while using a lower bitrate decreasing buffering.

Here is the
[developer guide](https://developers.google.com/youtube/v3/live/guides/hls-ingestion)
for HLS live streaming on YouTube for example. Adding better UI support for it in OBS will
make HLS live streaming with newer codecs much more accessible to end-users.

# Drawbacks

HLS ingestion has higher latency than RTMP.

# Additional Information

## UX Changes

-   Under Settings > Stream > Service dropdown list, add a new option for
    “YouTube - HLS”.
-   Display help text: HLS has slightly higher latency compared to RTMP, but is
    a protocol supporting next generation codecs that can reduce bandwidth
    requirements.
-   There will be two Server choices: “Primary YouTube HLS ingest server” and
    “Backup YouTube HLS ingest server”. The primary server is for primary
    ingestion and the backup server is for backup ingestion.
-   Users will copy the Stream Key from YouTube’s creator Studio similar to
    RTMP. See detailed instructions in Section 4 “Connect your encoder and go
    live” from
    “[Create a live stream with an encoder](https://support.google.com/youtube/answer/2907883?hl=en)”.
-   Rename the existing “YouTube / YouTube Gaming” option in the Service
    dropdown list to “YouTube - RTMP”.

## Implementation

There are two options for implementation: using the existing HLS live streaming
implementation in [FFmpeg](https://ffmpeg.org/ffmpeg-formats.html#toc-hls-2) or
implementing an HLS client from scratch as a new plugin. Using the existing
FFmpeg implementation via the
[FFmpeg muxer output](https://github.com/obsproject/obs-studio/blob/master/plugins/obs-ffmpeg/obs-ffmpeg-mux.c)
library is likely to be easier. The advantage of implementing from scratch is
that we may have more control of the parameters we want to set. However, the
parameters that we care about such as segment duration, playlist size and HTTP
user agent are already available as options in FFmpeg. So we will start with the
FFmpeg implementation, and write one from scratch if needed.
