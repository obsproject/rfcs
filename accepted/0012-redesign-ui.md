- Start Date: 2020-01-23
- RFC PR: #12
- Mantis Issue: none

# Summary

Redesign main UI to iron out the user experience.

# Motivation

Make it easier for users to find certain items. It would also modernize the look of OBS.

# Detailed design

## User UX

- Remove the controls dock. This would save space.
- Move the stream, record and replay buffer buttons to the status bar
- Add icons to these buttons
- Name the buttons Go Live, Record, Replay
- Redesign status bar
- In the top left corner: Create profile and scene collection combo boxes, so users can more easily change them
- In the top right corner:
	1. Add lock and visible checkbox for the preview
	2. Add checkbox for studio mode
	3. Add settings button
	4. Change the menu to a hamburger menu

# Drawbacks

Some users don't like change.
