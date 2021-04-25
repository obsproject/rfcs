- Start Date: 2020-03-05
- RFC PR: #13
- Related Mantis Issue: https://obsproject.com/mantis/view.php?id=1516

# Summary

Add the ability to output interlaced video.

# Motivation

TV broadcast is often interlaced, and Decklink cards offer the ability to output interlaced signals, but due to a limitation in OBS' output design, Decklink interlaced output is reduced to half frame rate progressive in an interlaced signal (i.e. 30p in 60i).

# Detailed design

## Technical considerations

- The output pipeline should be so that interlacing is applied after output scaling
- Output YUV shaders should account for interlaced chroma sample positions

## User UX

- Allow choosing top field first / bottom field first
- Add a global toggle to enable the option to select interlaced output in advanced options, to prevent unintentional use

## Other

- Potentially allow using a vertical lowpass filter to prevent flicker
- Devise a way to ensure deinterlaced sources preserve their field order to prevent outputting interpolated lines instead of original source lines
- Prevent field order swaps when OBS rendering drops a single frame (field)

# Drawbacks

Could be a pitfall for new users if they enable it without knowing what it does.

# Alternatives

Use ffmpeg custom output with -vf interlace
