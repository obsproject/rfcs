# Summary

I would like to propose an overhaul on the way Multiviews are handled and configured in OBS. A lot of the ideas here are coming from combining what I believe to be a lot of the best cases of multiviwer configurations from a lot of big switcher brands (Grass Valley, Ross, Blackmagic Design, etc) as well as trying to think differently in that we are a fully digital compositing system. In that regard, I've also heavily researched the EVS dyvi system as its heavily gpu-based. The aspect ratio of each multiviewer box would be assumed to match the canvas resolution, as it works currently.

![Here is my proposed example visual for how one would go about configuring a multiviewer.](https://i.imgur.com/I2S4GQ7.png)

The Proposal:
 * The control would be pulled out of the general settings tab and moved into its own settings tab to make it more free-form rather than having to conform to a dropdown.
 * There would be tabs for each multiviewer head, with a plus similar to a browser to create a new multiviewer head.
 * The layout would be built by adding rows first, with each row added prompting for how many columns would exist. For example if I want to add Preview and Program and nothing more, I would add a row then type in 2 columns to give me 2 half-width boxes.
 * All content of the multiviewer would exist centred on the output, so if you only have 1 row of 2 things, it would simply appear letterboxed a quarter of the way down the screen.
 * Boxes would be selectable, if a single box was selected a button would reveal itself to allow breaking out the box into 4 smaller boxes or unmerging that box. If multiple boxes were selected, you would only have a "Merge Boxes" button.

![Breaking out Example](https://i.imgur.com/7Rdntzm.png) ![Merging example](https://i.imgur.com/7RZB8x0.png)
 * The selection of what goes in each box would be handled via 2 dropdown menus. The first dropdown menu is populated with source types, and then the latter would filter all available sources by the source type, as Preview and Program are internal already, they could simply be denoted as "Internal" or "special" source type. This allows for quicker slotting in of not only scenes, but also regular sources meaning you could add a browser source, video capture device or game capture directly into the multiviewer.
 * Tally for the borders would be done by checking the source being active in Program or being in preview, whereas Program and Preview would never light up of course. Scenes would be checked in a similar fashion. This is beneficial as that way if you have a scene in program that has multiple things in it (like a gameplay overlay) then all active sources would light up red instead of (in the current system) just your active scene.

In terms of getting those outputs projected, I am unsure of having them in the View menu or from within each Multiviewer tab provides more merit one way or the other. I think either way is fine, as the actual set up of the Multiviewers is what I'm hoping to rework here. If they remain in the View menu, then can simply list each Multiviewer head and select display or Windowed out of the pop-up, if it exists on the settings modal then I imagine it'd be somewhat similar towards the bottom of the configuration pane, perhaps behind an "output" button. 

However, for the outputs themselves: *the labels for each box* should not have a background on it, and the text should be translucent. Only the Blackmagic switchers have backgrounds on the labels and the backgrounds make it extremely difficult to see what's going on behind it albeit existing in a location where usually important information just inside of title & action safe stuff would go (such as a timer for a countdown slate or most bottom-central UI elements in games). Almost all other major switcher lines make the labels translucent on top of video, or underneath/above the video box and I think we should avoid that as it makes the boxes less compact.

# Motivation

*What problem is this solving?*

The current multiviewer configuration is relatively lacking compared to other switcher solutions in the broadcast space. We only have 1 multiviewer head, and it is limited to only being able to look at scenes in the scene list top to bottom, which means if you want to re-order scenes in the multiviewer you also need to muck up your scene list order. Additionally, we are limited to very fixed presets for how that single multiviewer head is configured which leaves a lot to be desired for the in between scenarios. This proposal looks to make the multiviewer far more flexible, allow better tally tracking down to a source level, as well as trying to make it simplistic but still robust to create multiviewer layouts.

*What are the common use cases?*

Any level of "professional" use, as well as anyone that wants to be able to look at more than just preview and program. I think this will also help ease in users from vMix a bit since they'd be able to create 

# Drawbacks

What is the potential detriment for adding this feature/change?

This will make multiviewers more complicated out of the box to use for users. Perhaps a workaround to help with that, is by allowing some sort of sane default like the first 8 scenes being selected by default and providing a "Multiview 1" head with a relatively standard 2 (on the top of PVW, PGM) + 8 configured so that lesser experienced users can still easily access an out-of-the-box multiviewer while still providing all the flexibility that I think OBS should push towards in the long-term for more professional use cases.

# Additional Information


I know that the UI patterns are changing, but I wanted to be able to help provide a visual aid to the changes I had described above, and it should all be relatively easily flippable into the new stuff Warchamp and feaneron are working on.

For transparency, I don't know how much help I can be from a code-perspective on as I am more of a power-user than a developer but I will happily support however I can if someone else (or multiple people) would be willing to take up the helm on this.