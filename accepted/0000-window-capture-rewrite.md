- Start Date: 2019-11-06
- RFC PR: #1
- Mantis Issue: N/A

# Summary

Rewrite Window Capture on Windows to be 1) more consistent 2) more user friendly 3) don’t capture a random window as a fallback 4) better support for strange windows

# Motivation

Window Capture is one of the core capabilities of OBS, and yet its behaviour can be wildly unpredictable (and sometimes even downright dangerous). It has a number of flaws.

1. When a Window Capture source is initially created, it defaults to the first window in the list. If the user is currently streaming, this is bad - it can expose personal data because it’s unpredictable.

2. When a Window Capture source’s window is lost, OBS by default tries to find a “similar” window. However, “similar” can vary wildly depending on the application. Say you’re capturing a specific Chrome window. If OBS loses that window even for a second, it’ll try and find another Chrome window - one that may expose unwanted data.

3. When the user clicks “Cancel” without changing any settings, the Window hook is “refreshed”, causing a brief flash to viewers. The assumption is that if nothing has changed, *and* the user decided to “discard” the changes, nothing should change.
    - On a related note, the same behaviour doesn’t happen if the user clicks “OK” without changing any settings.

4. Minimised windows are not included in the Windows dropdown. While this may be a good way to exclude “hidden” windows, it can cause user confusion because the window *is* “open”.

5. The list of windows is not automatically refreshed, and provides no way for the user to manually refresh them.

6. The list of windows is not sorted in any meaningful way, making it hard to find a window, especially if the user attention is supposed to be elsewhere.

7. Windows that change their title often (eg. Dolphin emulator) can cause performance issues because OBS tries matching by title, which (sometimes) means unhooking and re-hooking, or repeatedly changing details about the hook unnecessarily.

I attempted to make some of these changes on my own, however due to the way the hooking loop works, I fear a chunk of it will need to be rewritten entirely to properly support all edge cases.

# Detailed design

I’ll start with the numbered points under “Motivation”, then add extras after that.

1. By default, Window Capture should default to “No window selected”. While this’ll be different than Display Capture defaulting to the first monitor (user can minimise sensitive windows beforehand), and Game Capture defaulting to “Any fullscreen game” (games are less likely to contain sensitive data), it makes sense for the capture type. This ensures that the user always manually selects which window is exposed.

2. Either allow the user to define the behaviour when a window is lost, or choose a sane default: freeze on the last good frame, or go invisible (the second is likely better feedback to the user).

3. Clicking Cancel should not flash the window. This could be that Cancel tries to “undo” all changes, and because nothing has changed, everything gets unset then set again, which causes the flash. Ideally, it should compare before/after and if everything is the same, do nothing.

4. Minimised windows should be included in the dropdown, always. This is a pretty easy change on its face, but edge cases like hidden minimised windows should be tested.

5. The list should refresh when the user opens the dropdown, and there should be a simple “Refresh” button near the dropdown.

6. Windows should be sorted alphabetically (by window title, not exe).
    - The dropdown should have “search/filter” capabilities
    - The user should be able to manually type in the name of the window, in the rare cases where they know the window title in advance and would rather Window Capture always capture that window
    - The user should be able to drag an icon from the Properties window onto the window they want to capture, instead of finding it in the list. “It’s right there, I just want to capture it”

7. Users have suggested a “Regex” or “Partial match” mode, where they include the non-changing part of the window title, and if that window isn’t lost, don’t try and rehook. Alternative solutions are welcome.

# How We Teach This

N/A, this simply changes the current behaviour. As long as the patch notes properly describe the changes, we should be fine.

# Drawbacks
This does change default behaviour, and *may* invalidate some weird edge cases I haven’t thought of. If it does, please mention them below so that we can ensure this RFC covers more use cases.

Some users struggle when a source like Game Capture doesn’t immediately capture the window they want - they assume OBS “just knows”. Currently, OBS captures the first window it can find, which incentivises the user to choose the actual window they want. By defaulting to a blank option, the user may be unsure what to do.

# Alternatives
Selecting the window *before* actually creating the source may be a viable feature. This would require changing the flow somewhat - the Add menu/dialog could list windows for the user to capture, and then the source could be created with the window already selected. This bypasses the drawback of potential user confusion about a window not being selected by default, and solves the “random default” issue too.

# Unresolved questions

N/A
