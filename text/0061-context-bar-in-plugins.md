# Summary

<!-- Simple explanation of feature/changes -->

Move the source-type specific implementation of the context-bar into plugins.

## Implementation in libobs

Add a new callback into `struct obs_source_info`.
```c
	obs_properties_t *(*get_short_properties)(void *data, void *type_data);
```
This callback will return the property information of this source for the context bar.
Unlike the existing callbacks `get_properties` and `get_properties2`,
it is expected to return a few subset of the properties.

Add a new API `obs_source_short_properties` that returns the property list for the context bar.
If `obs_source_info.get_short_properties` is not defined, the API will return NULL.
The interface is the same as the existing API `obs_source_properties`.
```c
obs_properties_t *obs_source_properties(const obs_source_t *source);
obs_properties_t *obs_source_short_properties(const obs_source_t *source);
```

The context-bar version of `obs_get_source_properties` won't be implemented
since the context-bar will be created only when there is an instance.
```c
obs_properties_t *obs_get_source_properties(const char *id);
```

## Alternative implementation in libobs

Add new callbacks `obs_property_set_context_bar` and `obs_property_context_bar` as below.
These callbacks will set and get each property that should be available for the context bar.
```c
void obs_property_set_context_bar(obs_property_t *p, bool enabled);
bool obs_property_context_bar(obs_property_t *p);
```

In the callback `get_properties` or `get_properties2` in the plugin should call `obs_property_set_context_bar` to indicate which properties should be shown in the context bar.

The drawback of this alternative implementation is that the properties of the context bar should be fully subset of the properties of the original properties.
For example, though the browser source has a button `Refresh cache of current page` in the property dialog, the context bar has a short text on the button `Refresh`. This difference will be covered by a different description returned by `get_short_properties`.


## Implementation in UI

The `class WidgetInfo` displays one property and handles its value and change.
The class is currently expected to be instantiated from the `class OBSPropertiesView`.
In the new implementation, the `class WidgetInfo` will be renamed to `class OBSPropertyWidget` and it will be generalized so that the class can be instantiated from the context bar.
Major changes in the class will be changing direct access to the members of `class OBSPropertiesView` but will emit signals, which will be connected to slots on the parent instance.

For the filter properties, the undo and redo handler is pushed into the stack using a 500-millisecond timer.
The timer will be moved into `class OBSPropertiesView`.
Hopefully, this change will fix [the minor bug on the redo](https://github.com/obsproject/obs-studio/issues/10628).

## Implementation in plugins

All these plugins will implement the callback `get_short_properties`.

- `browser_source`
- `wasapi_input_capture`, `wasapi_output_capture`, `wasapi_process_output_capture`
- `coreaudio_input_capture`, `coreaudio_output_capture`
- `pulse_input_capture`, `pulse_output_capture`
- `alsa_input_capture`
- `window_capture`
- `xcomposite_input`
- `monitor_capture`
- `display_capture`
- `xshm_input`
- `dshow_input`
- `game_capture`
- `image_source`
- `color_source`
- `text_ft2_source`
- `text_gdiplus`

# Motivation

<!-- What problem is this solving? What are the common use cases? -->

UI has the implementation of the context-bar for each source type.
As a result, property names are hard-coded in both UI and plugins.
In possible future modification, it might require to change both plugin and UI, which might lead to a potential bug.

It is a good implementation to separate the plugin-specific implementation from the UI.

This will also enable 3rd party plugins to have their context bar.

# Drawbacks

<!-- What is the potential detriment for adding this feature/change? -->

This makes it unable to have features that are not supported by `obs_property_t`.
So far, I don't see any features on the context bar that cannot be covered by this RFC.

# Additional Information

<!-- Any additional information that may not be covered above that you feel is relevant. External links, references, examples, etc. -->

Preliminary implementation in libobs is as below.
https://github.com/norihiro/obs-studio/commit/8ffc569f13993af317b2917bfade409cf6d02799
