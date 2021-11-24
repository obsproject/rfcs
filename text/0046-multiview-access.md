- Start Date: 2021-11-24
- RFC PR: #46

# Summary

Allow plugins, such as NDI and Decklink output to use the multiview as an output.

# Motivation

* Currently, the only way to output the multiview, is via the projector output. Users might want to use other ways to display the multiview.
* It would also allow each multiview to have its own settings, such as layout and sources

# Detailed design

## User UX

* For the Decklink output, add another section under "Preview Output" called "Multiview"
* Do the same for other similar plugins

## Backend

* The multiview code would be ported to libobs
* Create obs_multiview_t struct

* Functions:
```
- Used to create multiview with desired settings
  obs_multiview_t *obs_multiview_create(obs_data_t *settings)

- Similar to obs_source_update() function
  void obs_multiview_update(obs_multiview_t *multiview, obs_data_t *settings)

- Renders the multiview
  void obs_multview_render(obs_multiview_t *multiview)

- Increments reference count
  void obs_multiview_addref(obs_multiview_t *multiview)

- Decrements reference count
  void obs_multiview_release(obs_multiview_t *multiview)

- Destroys multiview when all references are released
  void obs_multiview_destroy(obs_multiview_t *multiview)

- Gets source by x, y coordinates in multiview
  obs_source_t *obs_multiview_get_source_by_pos(obs_multiview_t *multiview, int x, int y)

- Adds source to multiview
  void obs_multiview_add_source(obs_multiview_t *multiview, obs_source_t *source)

- Removes source from multiview
  void obs_multiview_remove_source(obs_multiview_t *multiview, obs_source_t *source)
```
