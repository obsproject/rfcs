# Summary

This RFC proposes the addition of libobs and various object (sources, encoders, etc.) APIs that allow querying system resource usage.

# Motivation

When using larger scene collections with many sources it can be difficult to identify which elements use system resources.

For example:
- RAM and CPU usage of browser sources
- VRAM usage of image sources or delay filters
- CPU usage of x264 and other CPU encoders
- Network usage of a media source streaming in data

# Proposed implementation approaches

## 0. General additions

The platform utilities shall be extended with APIs to query CPU/RAM/etc. usage of individual threads or processes that are not OBS itself (e.g. browser subprocess, encoder thread), as well as other information that may be necessary to implement reporting in plugins.

## 1. Resource-specific APIs

### libobs APIs

Additional APIs for querying specific resource usage would be added to all applicable source types, for example:

- `int64_t obs_encoder_get_cpu_usage(obs_encoder_t *source)`
- `int64_t obs_source_get_system_memory(obs_source_t *source)`

Non-support would be indicated by a predefined invalid return value (e.g. `-1`).

### Source/Encoder/etc. APIs

The "info" struct (e.g. `obs_source_info`) of a given object would be extended by APIs for querying usage of a specific resource, e.g.

- `info.cpu_usage()`
- `info.system_memory()`

The capabilitiy of a source would be determined by the absence or presence of it in the info object registered with OBS.

## 2. Capability flags and structs

In this case rather than adding specific APIs for resources sources would be queried to fill in a struct containing fields for all the values supported by libobs, and indicate via flags which are supported and shall be read by the consumer.

*Possible alteration: Flags could be part of the returned struct itself rather than be queried separately.*

### libobs APIs

A new struct would be introduced, e.g. `obs_resources_t` with a structure similar to this:

```c
struct obs_resources {
	float cpu,
	float ram,
	float vram,
	...
}
```

Additionally, new flags would be added to allow objects to indicate which values can be queried, e.g.
```c
enum supported_queries {
	OBS_RESOURCE_RAM = 1 << 0,
	OBS_RESOURCE_CPU = 1 << 1,
	...
}
```

An API user would then query the information from an object via its specific resource query API, as well as the supported flags to determine its supported values. All other data shall be discarded.

For example:
- `uint32_t obs_get_source_resources_flags(const char *id)`
- `uint32_t obs_source_get_resources_flags(obs_source_t *source)`
- `obs_resources_t *obs_source_query_resources(obs_source_t *source)`

### Source/Encoder/etc. APIs

The info object would be extended to add additional flags and one method to query resource usage:

```c
struct obs_source_info my_source = {
...
	.resource_query_flags = OBS_RESOURCE_RAM | OBS_RESOURCE_CPU,
	.resource_query = my_source_resource_usage,
...
```

The resource query function could have a signature like `void resource_query(void *data, obs_resources_t *resources)`.

# Drawbacks

In order for resource usage to be shown this must be implemented by sources and other eligible objects. This may not be trivial especially for GPU/VRAM usage.

Additionally there may be a performance hit from data collection, as such it should only be enabled when a user is actively monitoring usage (e.g. has an OBS internal "task manager" open).

# Additional Information

- Chrome developer API with similar goals: https://developer.chrome.com/docs/extensions/reference/processes
