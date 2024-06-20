# Summary

This RFC intends to facilitate discussion to find a way to move away from storing codec identifiers as strings. This is to avoid string comparisons and remove some redundancies such as outputs defining their own codec enumerations.

# Motivation

String comparisons are slow, error prone (case sensitivity, typos, etc.), and generally ugly. Most outputs convert from strings to internal enumerations anyway which could be made redundant by providing them in the first place.

# Overview

There are two components to this RFC: Codec Identifier and Codec Tags.
The former specifies a codec family such as H.264/AVC, while the latter specifies a specific variant used for the encoded bitstream.

## Codec Tags

Codec "tags" or FourCC codes (`uint32_t`) are used by containers such as MOV/MP4 or Enhanced FLV, and are used by some codecs to identify specific flavours or variants.
For example, ProRes uses several different ones depending on the specific feature set used (ProRes LT/HQ/XQ/etc.).
Currently support for this is hacked in by passing a value through `obs_data` objects, but this should be implemented natively.

### Proposed API

Codec tags can change depending on encoder configuration, and cannot be a static value in the `obs_encoder_info` struct, thus there needs to be a function that can be called to ask the encoder for the appropiate tag for its current configuration.

- New API function: `uint32_t obs_encoder_get_codec_tag(obs_encoder_t *encoder)`
- New `obs_encoder_info` struct member: `uint32_t (*codec_tag)(void *data)`

For most codecs the implementation of the `codec_tag` function may be ommited and a default value is provided by libobs. This default tag could be obtained from libavcodec via `av_codec_get_tag()` or hardcoded to the ones we currently use (i.e. `avc1` for H.264, `hvc1` for HEVC, `av01` for AV1).

**Note:** FFmpeg currently has two separate codec tag lists, one for ISO-BMFF/QuickTime (MP4/MOV) and one for RIFF (e.g. AVI), this proposal would only implement the MOV/MP4 tags as the RIFF ones are not relevant to OBS outside of possibly DirectShow.
We may elect to remove ambiguities by having a second paramter specifying a codec tag namespace, or explicitly naming the function to indicate that it returns ISO format tags.

## Codec Identifier

This RFC proposes three different methods, one of which should be agreed upon:

### FFmpeg AVCodecID

This approach would simply use `AVCodecID` enum defined in libavcodec.

#### Advantages

- libobs already links against libav* libraries so IDs can be used without many changes
- Easy to use in FFmpeg muxer and FFmpeg-based outputs (e.g. SRT/RIST)
- libav* functions available for converting from current string-based names to FFmpeg IDs

#### Drawbacks

- Not all modules that interface with video data or implement encoders currently depend on ffmpeg (e.g. obs-outputs, VideoToolbox)
- While unlikely, it could happen that FFmpeg does not support a codec we want to support, or support is missing from an older version, e.g. on LTS releases

### Codec Tags / FourCC

In this case we define the aforementioned codec tag as the new identifier.

#### Advantages

- Avoids a secondary API for codec identifiers
- FFmpeg's ISO/QuickTime tag list should support all codecs upstream OBS supports
- No external dependencies required

#### Drawbacks

- Many codecs have more than one tag, which could require large if clauses and may still require mapping to an internal enumeration value
- Codec support for ISO and QuickTime formats is limited
- Tags are mostly intended to be opaque values that are relevant to the muxer, and probably shouldn't be used as a unique identifier for a codec family

### OBS-specific IDs

In this case we would define our own enumeration, for example:
```c
enum OBSCodecID {
    OBS_CODEC_INVALID = 0,
    OBS_CODEC_H264,
    OBS_CODEC_HEVC,
    OBS_CODEC_AV1,
    OBS_CODEC_PRORES,
}
```

#### Advantages

- Simple enum in libobs
- No external dependencies

#### Drawbacks

- Requires libobs API changes to add new codecs
- Would prevent third-party plugins from adding codecs not supported by upstream OBS

A potential workaround is to still have a fallback such as `OBS_CODEC_CUSTOM` which still allows passing a string to the muxer, which would need special support for handling codec string names which is probably undesireable.

# Additional Information

- FFmpeg Codec Defintions: https://github.com/FFmpeg/FFmpeg/blob/0ae157b3603f27d8057febd8f2680ac1030722ee/libavcodec/codec_id.h#L49
- RFC 6381, specifying the common use of ISO codec identifiers for MIME types: https://datatracker.ietf.org/doc/html/rfc6381
