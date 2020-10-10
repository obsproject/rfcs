# Summary

Add HLS output as one of the live streaming output options in OBS.

# Motivation

[HLS](https://tools.ietf.org/html/draft-pantos-hls-rfc8216bis-07) live streaming is supported by a number of platforms. It is a generic output method. FFmpeg has an HLS option as a muxer output: https://ffmpeg.org/ffmpeg-formats.html#hls-2. If a URL is specified, it will stream the output to the destination. Akamai has a [specification](https://learn.akamai.com/en-us/webhelp/media-services-live/media-services-live-encoder-compatibility-testing-and-qualification-guide-v4.0/GUID-6A14ED6D-0A23-4122-AB60-64A49B6628B5.html) for HLS ingestion as well. Many hardware encoders support HLS output. YouTube also supports HLS ingestion for live streaming and its [specifications](https://developers.google.com/youtube/v3/live/guides/hls-ingestion) are compatible with others.

HLS can support next generation codecs that RTMP does not. Newer codecs can offer much better compression relative to
H.264, allowing users to stream with higher quality for a given bitrate, or stream with the same quality while using a lower bitrate decreasing buffering.

Currently one can do HLS output using OBS by setting FFmpeg HLS option in the recording output settings and starting to record. This is a bit convoluted though. So the goal of this RFC is to make the HLS output option more user friendly and more accessible.

# Drawbacks

HLS output has higher latency than RTMP.

# Additional Information

## UX Changes

Following the convention of the current settings for RTMP output, we will add an option for "YouTube - HLS". Note that the changes made by this RFC will lay the groundwork for adding a generic HLS output service option. The "YouTube - HLS" option just has the YouTube specific settings such as server URLs etc.

-   Under Settings > Stream > Service dropdown list, add a new option for
    “YouTube - HLS”.
-   Display a "More Info" button next to the service dropdown list if "YouTube - HLS"
    is selected. The button leads to https://developers.google.com/youtube/v3/live/guides/ingestion-protocol-comparison.
-   There will be two Server choices: “Primary YouTube ingest server” and
    “Backup YouTube ingest server”. The primary server is for primary
    ingestion and the backup server is for backup ingestion.
-   Users will copy the Stream Key from YouTube’s creator Studio similar to
    RTMP. See detailed instructions in Section 4 “Connect your encoder and go
    live” from
    “[Create a live stream with an encoder](https://support.google.com/youtube/answer/2907883?hl=en)”.
-   Rename the existing “YouTube / YouTube Gaming” option in the Service
    dropdown list to “YouTube - RTMP”.    
![YouTube - HLS service option](https://user-images.githubusercontent.com/233044/85955326-c1016000-b94b-11ea-8781-6027768c629b.png)

## Implementation

FFmpeg already has HLS as a muxer output option: https://ffmpeg.org/ffmpeg-formats.html#hls-2, so we can use the existing
FFmpeg implementation via the [FFmpeg output](https://github.com/obsproject/obs-studio/blob/master/plugins/obs-ffmpeg/obs-ffmpeg-output.c) plugin. The options for HLS output in FFmpeg are very comprehensive including segment duration, playlist size and HTTP user agent etc., which should be able to meet the requirements of most services accepting HLS ingestion.

For this RFC, we are going to add support for HLS with H.264. This will lay the groundwork for adding support for additional codecs in the future. 
