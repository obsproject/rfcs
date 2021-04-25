- Start Date: 2020-09-10
- RFC PR: #32

# Summary

This proposal is for adding a scene wizard for new users when first loading OBS.

# Motivation

Users often find it difficult to set up OBS.

# Detailed Design

The user will also be able to open the wizard in the scene collection menu.

### Wizard page 1
- Buttons that allow the user to select how they want to use OBS
	1. "I just want to record my screen"
	2. "I want to use a scene template"
	3. "I will set up OBS myself" (if they select this then exit wizard)

- The scene templates would be just pre-made scene collections

### Wizard page 2
- If the user selects scene template, show them different scene collections they could choose from
- The scene collections will have image previews as well
- If the user selects screen recording, ask them what monitor they want to record

### Wizard page 3
- Have the user select their microphone and webcam
- The webcam would have a preview
- The microphone would also show a volume meter

# Additional Information

- Should it be a seperate wizard than the auto-config? Or combined into a single wizard?
