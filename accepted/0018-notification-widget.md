- Start Date: 2020-04-17
- RFC PR: #18
- Mantis Issue: N/A

- Related PRs:
	1. https://github.com/obsproject/obs-studio/pull/1832

# Summary

Add the ability to display notifications to the user.

# Motivation

Currenty, there is no way of showing custom notifications to users.

# Detailed design

## User UX

- Display notifications at top of main window
- Have three notification types (Info, Error, and Warning)
- Each type will display a different icon and the widget would be a different color for each
- Display custom text
- Add button for different types of actions (open source properties, open settings window, open url)
- The button's text will be customizable
- The widget will be floating over the window so other widgets aren't resized when it is displayed
- Add api to libobs so plugins can display notifications, instead of just the frontend

# Drawbacks

Users might find the notifications annoying.

# Alternatives

Display warnings in status bar.
