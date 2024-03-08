- Start Date: 2022-09-28
- RFC PR: #52

# Summary

Add an output manager so that outputs can be managed in a single dialog.

# Motivation

- Each output (NDI, Decklink, AJA, etc.) doesn't have to write their own UI, removing much deplicated code.
- It would make it easier to find bugs and fix them for all outputs instead of just one.

# Backend

- Create an output object wrapper class to simplify creating outputs
- This class will utilize the new video mixes feature

# UI

- The output manager should reside in the tools menu
- The UI should be similar to the filters dialog
- The reason for it to be like this dialog, is that users are already familiar with it
- Like the filters dialog, the list of the current outputs would be in a list view, with the eye toggle to turn each output on or off
- The properties of each output would be on the right side as well
- Clicking the plus icon will show the list of outputs to choose from (like the + button for sources, filters, etc.)
- Along with the properties on the right side, there should be a widget at the top in a split view to edit output type
- Widget specifics:
	1. A dropown to select output type (source, scene, multiview, preview or program)
	2. If source or scene, show another dropdown with the list of all sources or scenes to choose from
	3. A checkbox to select whether to auto start the output, when starting OBS
- This dialog could also be used for iso recordings
- This would be accomplished by adding a second list view for recordings, like how the filters dialog has two lists for async and effect filters

# Drawbacks

- How would existing outputs be migrated to this new system?
- Would the users have to reconfigure their existing outputs?
