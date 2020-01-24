- Start Date: 2020-01-23
- RFC PR: #11
- Mantis Issue: https://obsproject.com/mantis/view.php?id=602

- Related PRs:
	1. https://github.com/obsproject/obs-studio/pull/1321

# Summary

Add the ability for users to create global scenes thats are on top of every scene.

# Motivation

Users won't have to create meta scenes and copy them to every scene.

# Detailed design

## User UX

- Have an extra dock for the DSK scene (same as the scenes dock)
- Have the dock on by default, so users easily know about it
- Users could switch between DSK channels with combo box in the dock
- Alternatively, use hotkeys to change DSK channels, so we don't clog up the UI
- There would be an extra option in the combo box for disabling the DSK
- Call the DSK "Global Scenes" so users would more likely know what it is

## Other

- PR only implemented single DSK channel
- In the PR, users could disable the DSK with hotkeys
- PR needs to be refactored to remove libobs usage

# Drawbacks

Users might not know what a DSK is.

# Alternatives

Add an existing scene to scenes.
