- Start Date: 2020-04-21
- RFC PR: #22
- Mantis Issue: N/A

# Summary

- Add generic outputs dialog so all outputs can be accessed at once, instead of being in the tools menu.
- This would be similar to the advanced audio dialog

# Motivation

- With each output currently, developers have to create UIs for each one.
- These UIs may be hard to find in the tools menu
- This would simplify the output process and be more user friendly.

# Detailed design

## User UX

- Display name of output in first column
- Have a controls button to start and stop the output
- An auto-start checkbox to start the output when OBS is started
- Properties button to access the output properties
	1. The properties dialog would be similar to the source properties dialog
- Trash button to remove output
- At the buttom have a plus button
	1. The plus button would be used to add outputs, such as if the Blackmagic source allowed outputs for specific sources
	2. The plus button would bring up a menu similar to the add source menu

## How to teach this

- Put access to dialog in an easy to find place, such as in the controls dock.

# Drawbacks

- How to deal with existing plugins, that already have an UI.

# Alternatives

- Keep everything in tools menu
- Add buttons for each output in controls menu. This would probably get cluttered fast.
