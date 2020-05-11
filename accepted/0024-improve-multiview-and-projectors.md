- Start Date: 2020-05-08
- RFC PR: #24
- Mantis Issue: N/A


# Summary

Improve the projectors and the multiview window.

# Motivation

The projectors an the multiview windows are quite useful but do not provide much options for customization. 

# Detailed changes

### Changes in the multiview window

Add more freedom to customize the multiview:

- Add some sort of configuration window which allows the user to set the following settings:
  - number of rows 
  - number of columns
  - size of the preview in columns and rows
  - show/hide the preview
  - show/hide the program
  - show/hide the name of the scene/source
  - always on top
- show/hide safe areas (EBU R 95) for the preview and scenes/sources

- Add a right click menu to select what scene or source (including the preview and program) should be displayed in a cell of the multiview
- Add an option to transition to the scene by double clicking on one cell

### Changes in the projectors

- Add a right click menu entry to enable safe areas (EBU R 95) for a projector
- Add a right click menu to show/hide the source or scene name in the projector window
- Add an option to transition to the scene by double clicking on the projector
- Add a right click menu to select an exact pixel size of the projector windows. 
  This allows users to preview the scene/source on a scaled down resolution.

### Changes of projectors and multiview

- Provide an option in the settings to hide/show the projectors from the taskbar. 
  Having multiple projectors opened at the same time clutters the taskbar. It needs a second look to distinguish the main window from the projectors.
- Provide an option in the settings to hide/show the window frame. 
  When several projectors are aligned right next to each other the frame takes up screen space. 
If no window frame is activated, the window should be movable when dragging the the window content
- Make the projector and multiview windows dockable.
  Docking the projectors in the main window would allow to customize the UI of OBS even more.
- Add a right click menu to enable/disable "always on top" per projector or per multiview window



# Drawbacks

Too many options for customization could clutter the UI and confuse the user.

Depending on how the projectors and multiview(s) are persisted, migrating from an older Version to a newer Version could lead to loosing the saved projector/multiview settings.

# Additional Information

There are some ideas on the ideas page which are addressed by this RFC:

- https://ideas.obsproject.com/posts/667/borderless-windowed-projectors
- https://ideas.obsproject.com/posts/464/make-windowed-projectors-dockable
- https://ideas.obsproject.com/posts/250/windowed-projector-sizing-options



The multiview window and projectors are internally implemented within the same class. When implementing this RFC the original OBSProjector could be refactored to a OBSProjector base class and a derived OBSMultiview class. This would follow the separation of concerns principle and clean up the code.

Moreover the multiview code could be encapsulated to make it easy to replace the preview in the main window with it in the future.

