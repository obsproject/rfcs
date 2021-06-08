# Summary
Adopt the [Reverse Domain Name notation](https://en.wikipedia.org/wiki/Reverse_domain_name_notation) naming convention to reduce potential collisions.

# Motivation
Unified naming conventions reduce the chances of name collisions in first-/second- and third-party additions/changes.

# Drawbacks
- This may break existing setups unless a transition layer is implemented to transfer old to new identifiers.
- A verification layer may be necessary to enforce this rule, enforced on any plugin built against versions of libOBS that include this RFC.

# Design
Adopt the [Reverse Domain Name notation](https://en.wikipedia.org/wiki/Reverse_domain_name_notation) naming convention for every name-able object in the OBS API, for both internal and external additions. This reduces the chances of name collisions between external and internal plugins from new developers not well versed in OBS's internal structure, as well as provide necessary unique ids to plugins for future plugin managers - without the need of a different UUID.

## Example: OBS Internals
As an example, here is how the OBS internal additions would be named when this RFC is adopted:

* Sources
    * Audio In/Out Source (WASAPI): `com.obsproject.source.audio.wasapi.in` or `com.obsproject.source.audio.wasapi.out`
    * Browser Source: `com.obsproject.source.browser`
    * Color Source: `com.obsproject.source.color`
    * Display Capture: `com.obsproject.source.displaycapture`
    * ...
* Filters
    * Apply LUT: `com.obsproject.filter.lut`
    * Chroma Key: `com.obsproject.filter.key.chroma`
    * Color Key: `com.obsproject.filter.key.color`
    * ...
* ...

## Example: StreamFX
An example of a third-party integration:

* Sources
    * Source Mirror: `com.xaymar.streamfx.source.mirror`
    * Shader: `com.xaymar.streamfx.source.shader`
* ...

# Migration
There are two cases that need to be handled by the migration code:

1. When the entire plugin name changes to the RDNS naming convention.
2. When parts of the plugin change to the RDNS naming convention.

Both can be handled, though they do add overhead.

## 1. Plugin name changes
Plugins should expose a function that informs libOBS about old names of the binary, which is called before performing any further loading of the plugin. libOBS internally keeps a hash map of old to new names, which prevents loading duplicate binaries. Additionally libOBS could use the obs_module_ver information to only load the newest binary.

```
/** Get the previous names of this plugin. (Optional)
 *
 * @param names An array of 'const char*' UTF-8 strings of every previous name of the plugin.
 * @return The number of elements in the array.
 */
MODULE_EXPORT size_t obs_module_previous_names(const char*** names) {
    names = nullptr;
    return 0;
}
```

## 2. Feature name changes
This can be handled in two ways, both with different complexity and flexibility:

### 2.1 Proxy Table from registered elements
All objects to be registered expose an optional function to return previous identifiers of the source. This function is called when `obs_..._register` is called to build an internal map of old ids to new ids, which is only used when the current id of an object is not found in the registered ids.

```
struct obs_source_info {
    ...

    /** Get the previous names of this object. (Static, Optional)
    *
    * @param names An array of 'const char*' UTF-8 strings of every previous name of the plugin.
    * @return The number of elements in the array.
    */
    size_t previous_ids(void * type_data, const char** *ids);
}
```

### 2.2 Plugin Migration Function
As shown in 1, we can let the plugin itself handle id conversions. The function is only called if present and only if an id can not be found in the available object ids,

```
/** Migrate object ids. (Optional)
 *
 * @param old_id The original id of the object.
 * @param new_id The new id of the object, if the old_id was part of this plugin.
 * @return Returns true if the old_id was part of this plugin and the new_id was modified, otherwise false.
 */
MODULE_EXPORT bool obs_module_migrate_object_id(const char* const old_id, const char** new_id) {
    return false;
}
```

# Verification
The verification layer is an optional addition, which should prevent newer plugins from loading or registering objects if they fail the naming convention check. A plugin is considered "new" if its obs_module_ver exceeds a certain threshold, with the threshold being defined at implementation time. Plugins that do not match the "new" condition should only be warned about if they mismatch the verification requirements.
