# Summary

Add sRGB to "Color Space" output options, and default to sRGB.

# Motivation

There is often confusion around the "Color Space" option for output, and it has several problems.

- We don't actually convert colors between RGB color spaces in OBS. We merely tag the metadata.
- Rec. 601 output support is weird. x264 tags 601 as "undef" for everything, and jim-nvenc uses the European 625-line variant of 601 attributes.
- Any PC applications/games that output sRGB are tagged as 601/709, and will be handled incorrectly by color-accurate applications, e.g. old versions of Chrome handled 709 properly before they decided to cheat to save power.

Defaulting to sRGB is probably the least of all evils.

- sRGB is identical to Rec. 709 except for the sRGB transfer function, which can be a free conversion for most GPUs. When represented as YCbCr, it makes sense to transform with Rec. 709 matrix coefficients.
- PC applications/games that use sRGB (very common) are now tagged accurately if using sRGB. Video capture feeds that use 601/709 would not be, but we can leave behind 601/709 settings for passthrough scenarios. Actually converting the color data is beyond the scope of this RFC.
- Chrome supports sRGB videos properly on all tested platforms.
- Rec. 601 for output was almost always the wrong choice.

Implementation:

- Add VIDEO_CS_SRGB, and plumb into all switch cases.
- Go through output encoder settings, and use sRGB metadata values.

# Drawbacks

People that had a working color pipeline with the old faulty settings may have to tweak their setup.

There is supposedly a performance regression involving DeckLink when switching away from 601.

# Additional Information

None.