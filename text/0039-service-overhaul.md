# Summary

- Make OBS able to accept third-party service plugins
- Register services with a unique id rather than a common one
- A service can be provided with multiple protocols
- Separate Twitch, Restream and YouTube integration and make them plugins

# Motivation

Actually even if OBS has the Service API, developer can't create third-party service plugin because there is no mechanism to use them at all.

Before in OBS in the Stream settings page, property views were used for services but with only two registered services `rtmp_common` which contain all services and `rtmp_custom` for custom servers.

Currently in OBS, this page shows the list of services with many new elements shown or hidden depending on the selection (like recommended settings). With also Twitch and Restream OAuth integrations. And no use of the property views provided by `rtmp-services`.

This need to be refactored to re-introduce property views for the service and for the protocol output.

This will also provide the ability for some stream services to be able to make their own plugin.

# Design

## Settings UI

Many elements of the settings UI related to services will be moved inside services properties views like:
- Stream key field
- Username and password fields
- OAuth connect disconnect button
- Text with clickable link (e.g., YouTube integration links)
- Maximum and recommended settings information
- "Get Stream Key" button
- "More Info" button

The streaming output will be selected through the settings windows and its property view will be placed below the service properties view.

The service output settings (if any) will be saved as a JSON string in the profile config file.

Advanced network settings will also be replaced by the output properties view below the service properties view, since those are meant for `"rtmp_output"` (RTMP(S) output).

### Common, uncommon

Spoiler: Streaming services will be individually registered, so no more `rtmp_common` that represent various services.

So to hide services behind the "Show All" option, a flag (`OBS_SERVICE_UNCOMMON`) meant for "first-party" services marked as "uncommon" will be added to the Service API to indicate that the service is "uncommon".

So third-party services are shown in the list by default.

### Maximums and recommended

Maximums and supported resolutions will be strict (e.g. Twitch allows 6000 kbps, more will not be accepted), the UI will not allow to save incompatible settings.

This will require to introduce in the Property API a way to indicate that the settings are not valid.

TODO: Think about recommendation

## Frontend API

Those functions will be modified:
- `obs_frontend_set_streaming_service()` because while the output selection is done in the settings windows and is no longer done while the stream is starting will now swap the output if the service protocol and the actual output do not match.
- `obs_frontend_save_streaming_service()` will save service output settings.

## About the FTL protocol

The FTL protocol is actually planned to be deprecated once a new protocol is added after this RFC is implemented.

Any services that only support FTL will follow the deprecation cycle and be dropped at the same time as FTL if those services did not added support for another protocol.

## Service plugins
### Service API

Adding to `obs_service_info`:
  - `uint32_t flags` with the following flags:
    - `OBS_SERVICE_DEPRECATED`: The service is marked as deprecated.
    - `OBS_SERVICE_INTERNAL`: The service is meant to be used internally in some plugin (e.g., WebSocket, Scripting) and usually not directly exposed in the UI.
    - `OBS_SERVICE_UNCOMMON`: The service can be hidden behind a "Show All/More" option UI/UX-wise.
  - `const char *supported_protocols`: Protocol supported by the service.
  - `enum obs_service_audio_track_cap (*get_audio_track_cap)(void *data)`: Returns the service audio track capability with the following possible values:
    - `OBS_SERVICE_AUDIO_SINGLE_TRACK` - Only a single audio track is used by the service
    - `OBS_SERVICE_AUDIO_ARCHIVE_TRACK` - A second audio track is accepted and is meant to become the archive/VOD audio
    - `OBS_SERVICE_AUDIO_MULTI_TRACK` - Supports multiple audio tracks
  - `void (*get_defaults2)(void *type_data, obs_data_t *settings)`: Same as its non-2 variant but give access to the `type_data` pointer.
  - `obs_properties_t *(*get_properties2)(void *data, void *type_data)`: Same as its non-2 variant but give access to the `type_data` pointer.
  - `bool (*can_bandwidth_test)(void *data)`: Return if the service is able to do bandwith test
  - `void (*enable_bandwidth_test)(void *data, bool enabled)`: Enable bandwidth test on the service
  - `bool (*bandwidth_test_enabled)(void *data)`: Return if the service has the bandwidth test enabled
  - `void (*get_supported_resolutions2)(void *data, struct obs_service_resolution **resolutions,size_t *count, bool *with_fps)`: Replace its non-two variant to enable framerate value
  - `int (*get_max_video_bitrate)(void *data, const char *codec struct obs_service_resolution resolution)`: Return a maximum bitrate based on a video codec and a resolution
  - `int (*get_max_codec_bitrate)(void *data, const char *codec)`: Return a maximum bitrate for a specific codec

TODO: List functions added to the Services API

### `rtmp-services`

Services provided by this plugin (`"rtmp_custom"`, `"rtmp_common"`) will be deprecated (`OBS_SERVICE_DEPRECATED` and `OBS_SERVICE_INTERNAL` applied) and completely unused in OBS Studio.

Those are deprecated rather than completely removed to allow scripting and plugins to migrate if they happen to use them.

Its service JSON will no longer be updated.


### `custom-services`

If OBS Studio needs to be heavily dependent on one service plugin, it must be this one.

This plugin is meant to provide replacements for the `"rtmp_custom"` type.

#### `"custom_service"`

This the only service of this plugin that will be exposed to the user through the UI.

The protocol will be selected by the user and the properties view will change depending of the protocol.

This service will support all first-party protocols, the properties view will change visible fields according to the protocol.

And also support any third-party protocols by allowing to choose which fields (e.g., username + password, stream key, custom field…) to add in the view.

This service will be registered in the "post load module" step (`obs_post_load_modules()`) to make the service aware of all registered protocols in the "load module" step.

#### Custom type per protocol

Service types that support only the protocol inside their id (`"custom_`*insert lowercase protocol acronym*`"`) for any first-party protocol except FTL (deprecation).

Services that will only support the first-party protocol related to their id/type.
The flag `OBS_SERVICE_INTERNAL` will be applied to not show them in the UI list. Those are meant for being used in WebSocket and scripting.

### `obs-services`

This plugin is meant to provide a replacement for `"rtmp_common"` type for services who don't rely on custom behavior nor integration.

This plugin will need a brand new services.json with new format, to register each streaming service with their own id. No more things like `"rtmp_common"` id.
 
Those services will be able to provide multiple protocols so no more "Service - HLS" and "Service - FTL".

If a certain protocol is not available, the UI will not show it if the service doesn't support another protocol. Same goes for codecs.
If it does, the missing protocol will not be shown.

Those services should have no specific behavior like ingest management.

In the end, adding streaming service will only be a JSON addition for this plugin.
Only improvements will be accepted in the code of this plugin.

#### Service ID naming scheme
- Only lower case letter
- Hyphen `-` are only used to distinguish two service providing the same stream service but with noteworthy differences like NicoNico (free & premium).
- Like this if the service needs a specific behavior, you can 'transfer' it from `obs-services` to a new first-party plugin and keep the same id. The change will be seamless for the user if done properly.

### Plugin for special case services

This plugins is meant to provide replacements for `"rtmp_common"` type for services that were working around the JSON usually by adding ingest management code (except Twitch).

4 services are concerned:
- Dacast
- Nimo TV
- SHOWROOM
- YouNow (FTL only, so falls under the FTL deprecation cycle)

Beside the `"rtmp_common"` part being replace by a unique service type, most of the ingest code will stay identical.

This plugin is not meant to be maintained by OBS Project, but as a temporary measure until those services create their own third-party plugins. So all of those services will be put in deprecation.

### Conversion from old services

`streamService.json` will replace `service.json`, a new file approach is taken to avoid sudden breakage if the user happen to downgrade.

`streamService.json` will contain the type/id of the service and its settings but also the type/id of of the used output and its settings.

If `streamService.json` is not found, it will be generated from `service.json` if available.

A JSON or a harcoded list with old service name linked to their new id, to make OBS able to convert the service to a new one.

If the user had an no longer available service, the user will be switch to `custom_service`.

Integration settings will be migrated to their respective config file.

### Integrations

Twitch, YouTube and Restream have their own integration in OBS with OAuth. But those need to be isolated as plugins.

Those plugins will provide the basic and OAuth version of the services as one services.

If OBS is built without client ids and hashes, those plugins will be built without their integration.

#### Dock state

Actually the dock state is global (same on any profile). But with docks that appear depending on the service integration which is per profile.

This can lead to dock state losses between profile switching and exiting. Currents integrations actually store a dock state in the profile config and restore it after the integration is loaded.

But restoring a dock state from a plugin must not be considered at all.
Making it per profile could avoid this.

Integrations docks will be added after specific frontend event.
So after each of them the dock state needs to be restored.

- The initial restoration of the profile dock state will be moved just after `OBS_FRONTEND_EVENT_FINISHED_LOADING`. 

- `OBS_FRONTEND_EVENT_PROFILE_CHANGED`, profile dock state will be loaded after a profile is changes.

Also they will be removed after specific frontend event.
So before each of them the dock state needs to be saved.

- `OBS_FRONTEND_EVENT_PROFILE_CHANGING`, profile dock state will be saved before a profile is changed.

- `OBS_FRONTEND_EVENT_EXIT`, profile dock state must be saved before this event while OBS is exiting.

#### Dock addition

Integration plugins will need events/signals to be able to add and remove their docks at the right moment.

The plugin will have to store if `OBS_FRONTEND_EVENT_FINISHED_LOADING` or `OBS_FRONTEND_EVENT_PROFILE_CHANGED` was passed or not.

If created before these events, the plugin will wait their emission to add docks (if integration connected).

When created/updated (after these events) the service will add the docks (if integration connected).

When `OBS_FRONTEND_EVENT_PROFILE_CHANGING` `OBS_FRONTEND_EVENT_EXIT` or the service is destroyed, docks will be removed (if integration connected).

#### Browser features (MORE CHANGES INCOMMING)
The Front-end API needs to enable the possibility to access some `obs-browser` related feature like adding browser docks and generating widgets.

Note: Free functions for each structure that requires it because of a "dynamic" type will be also added.

`bool obs_frontend_browser_available()` will indicate if OBS Studio have the `obs-browser` included and available with a Wayland check on Linux/FreeBSD. This will allow plugins to know if they can use browser features.

```C++
struct obs_frontend_browser_params {
	const char *url;
	const char *startup_script;
	const char **force_popup_url;
  const char **popup_whitelist_urls;
  bool enable_cookie;
};
```

- `const char *url`, URL of the dock.
- `const char *startup_script` allow to set a startup script for the dock.
- `const char **force_popup_url` allow to set a list of URL to force those to popup.
- `const char **popup_whitelist_urls` allow to set a list of URL to popup whitelist.
- `bool enable_cookie`, if true `panel_cookie` will be set on the dock rather than a `nullptr`. This will allow to keep cookie which is required.

`void *obs_frontend_get_browser_widget(struct obs_frontend_browser_params *params)` is the function that will return a browser widget (QCefWidget) that we can cast to a QWidget for OAuth or a custom chat dock (e.g. Youtube).

`void obs_frontend_delete_browser_cookie(const char *url)` will remove cookies related to the given URL.

#### Broadcast flow
YouTube is not the only service that could have or need a "Manage Broadcast" button but the feature/flow is actually made only for how YouTube works.

So refactoring it to make service plugin able to use it without enforcing YouTube flow on them is required.

Inside the Frontend API:

 - `void obs_frontend_add_broadcast_flow(const obs_service_t *service, const struct obs_frontend_broadcast_flow *flow)` adds a broacast flow bound the given service. If the given service is actually the same service as OBS Studio use, the broadcast flow will be enabled.

 - `void obs_frontend_remove_broadcast_flow(const obs_service_t *service)` removes the broadcast flow. If the given service is actually the same service as OBS Studio use, the broadcast flow will be dsiabled.

```c
struct obs_frontend_broadcast_flow {
	void *priv;

	uint32_t flags;

	enum obs_broadcast_state (*get_broadcast_state)(void *priv);
	enum obs_broadcast_start (*get_broadcast_start_type)(void *priv);
	enum obs_broadcast_stop (*get_broadcast_stop_type)(void *priv);

	void (*manage_broadcast)(void *priv);
	void (*manage_broadcast2)(void *priv, bool streaming_active);

	void (*stopped_streaming)(void *priv);

	void (*differed_start_broadcast)(void *priv);
	enum obs_broadcast_stream_state (*is_broadcast_stream_active)(void *priv);

	bool (*differed_stop_broadcast)(void *priv);

	const char *(*get_last_error)(void *priv);
};
```

- `uint32_t flags`  with the following flags:
  - `OBS_BROADCAST_FLOW_ALLOW_MANAGE_WHILE_STREAMING`, the flow allow managing broadcast while streaming
  - `OBS_BROADCAST_FLOW_ALLOW_DIFFERED_BROADCAST_START`, the flow can set broadcast that require to be started after the streaming is started.
  - `OBS_BROADCAST_FLOW_ALLOW_DIFFERED_BROADCAST_STOP`, the flow can set broadcast that require to be stopped after the streaming is stopped.

- `enum obs_broadcast_state (*get_broadcast_state)(void *priv)` return the state of the broadcast:
  - `OBS_BROADCAST_NONE`, no broadcast is setup
  - `OBS_BROADCAST_ACTIVE`, the broacast is active
  - `OBS_BROADCAST_INACTIVE`, the broadcast will need a differed start

- `enum obs_broadcast_start (*get_broadcast_start_type)(void *priv)` return the start type of the broadcast:
  - `OBS_BROADCAST_START_WITH_STREAM`, the broadcast start with the stream
  - `OBS_BROADCAST_START_WITH_STREAM_NOW`, same as the previous but streaming is also started
  - `OBS_BROADCAST_START_DIFFER_FROM_STREAM`, the broadcast will need a differed start

- `enum obs_broadcast_stop (*get_broadcast_stop_type)(void *priv)` return the stop type of the broadcast:
  - `OBS_BROADCAST_STOP_NEVER`, the broadcast is not meant to be ended
  - `OBS_BROADCAST_STOP_WITH_STREAM`, the broadcast is ended with the stream
  - `OBS_BROADCAST_STOP_DIFFER_FROM_STREAM`,the broadcast will need a differed stop

- `void (*manage_broadcast)(void *priv)` and `void (*manage_broadcast2)(void *priv, bool streaming_active)`, callcack to create and show the service broadcast manager. The `2` variant allows to open it while streaming.

- `void (*stopped_streaming)(void *priv)`, signal to the flow that the broadcast has stopped allowing to change state if needed

- `void (*differed_start_broadcast)(void *priv)`, start the broadcast if differed. If the broadcast is not updated to active, the broadcast failed to start. If success OBS Studio will run a thread with the next callback to wait for the stream to be active.

- `enum obs_broadcast_stream_state (*is_broadcast_stream_active)(void *priv)` return the state of the broadcast stream:
  - `OBS_BROADCAST_STREAM_FAILURE`, stream has failed to be started
  - `OBS_BROADCAST_STREAM_INACTIVE`, stream is not started
  - `OBS_BROADCAST_STREAM_ACTIVE`, stream is started

- `bool (*differed_stop_broadcast)(void *priv)`, stop the broadcast if differred. Returns `true` if it succeeded

- `const char *(*get_last_error)(void *priv)`, return the last error if differed callbacks failed.

#### Twitch and YouTube bandwidth test
Those services have code in the UI to have such feature, it will now be moved to their respective services with the new Services API related to bandwidth test.

#### Twitch VOD track
This Twitch-only feature will be modified to rely on `obs_service_audio_track_cap` rather than the service name.

### If the saved service id is not registered becuase the plugin is longer there
OBS Studio will fallback to an empty `"custom_service"` with a message for the user explaining that maybe a plugin is missing.

## Service JSON for `obs-services`

### Old version from `rtmp-services`

The old actually looks like this:
```json
{
    "format_version": 3,
    "services": [
        {
            "name": "Example",
            "common": false,
            "more_info_link": "https://example.com/more_info",
            "stream_key_link": "https://example.com/stream_key",
            "alt_names": [
                "Example with a FTL server"
            ],
            "servers": [
                {
                    "name": "Server",
                    "url": "ftl://example.com"
                }
            ],
            "recommended": {
                "keyint": 2,
                "profile": "main",
                "output": "ftl_output",
                "max video bitrate": 4321,
                "max audio bitrate": 321,
                "supported resolutions": [
                    "1600x900",
                    "1280x720"
                ],
                "max fps": 24,
                "bframes": 2,
                "x264opts": "scenecut=0 tune=zerolatency",
                "bitrate matrix": [
                    {
                        "res": "1280x720",
                        "fps": 24,
                        "max bitrate": 3210
                    },
                    {
                        "res": "1600x900",
                        "fps": 24,
                        "max bitrate": 4321
                    }
                ],
            }
        }
    ]
}
```

- `"format_version"`: the format version, changed when the JSON is no longer backward compatible.
- `"services"`: array of services.
  - `"name"`: name showed in OBS.
  - `"common"` (optional): is the service only shown when the user has clicked on `Show all`.
  - `"more_info_link"` (optional): link with more info about the service.
  - `"stream_key_link"` (optional): link where the user can find his stream key.
  - `"alt_names"` (optional): alternative service name.
  - `"servers"`: array of servers.
    - `"name"`: name of the server.
    - `"url"`: url of the server.
  - `"recommended"` (it depends): "recommended" settings that become the default when using this service.
    - `"output"` (not optional if using a protocol different from RTMP(S)): indicates which output **shall** be used because the service use a protocol different from RTMP(S).
    - `"keyint"` (optional): default keyframe interval for this service.
    - `"bframes"` (optional): default b-frames for this service.
    - `"profile"` (optional): default H264 encoder profile for this service.
    - `"x264opts"` (optional): default option for x264 for this service.
    - `"max video bitrate"` (optional): maximum video bitrate for this service.
    - `"max audio bitrate"` (optional): maximum audio bitrate for this service.
    - `"max fps"` (optional): maximum frame per second for this service.
    - `"supported resolutions"` (optional): array of supported resolutions for this service.
    - `"bitrate matrix"` (optional): array of maximum bitrate for a certain resolution/FPS combo for this service.
      - `"res"`: resolution.
      - `"fps"`: frame per seconds.
      - `"max bitrate"`: maximum video bitrate.

#### Issues with this format
- About `"output"`, OBS consider every service as RTMP if not added and it's not a recommended settings at all. It makes OBS use the right protocol for the service. Also this prevent a service of being multi-protocol.
- About `"recommended"`, most of the options seems to H264 related and are mostly only maximums.
- `"common"`, the name makes it not understandable at the first sight maybe adding some documention would be good thing.

### New format (WIP)

The new format will rely on more strict JSON schemas.

Those JSON Schemas (Draft 2020-12) will be:
- `protocolDefs.json`: enums, regex patterns and properties related to protocols to ease protocol additions or removals in schemas
- `codecDefs.json`: enums, regex patterns and properties related to codecs to ease codec additions or removals in schemas
- `service.json`: define the service object itself, will be used for every service plugin that needs a JSON
- `obs-services.json`: `obs-services` JSON Schema

Those are all present in `0039-json-schemas` folder with an example of `obs-services` JSON.

*Documentation in schemas are in progress, and a schema for integration plugin should also be made.*

JSON Schema allows to validate `services.json` (in `obs-services` case) when a PR is made against it.

The use of JSON Schema allows to create specific additions for a specific plugin by creating a unique schema for it.

#### Advantages of this format

- Recommended settings are really recommended settings.
- Recommendation and maximums are two separated thing.
- Some settings are now per protocol or per codec, this allow multi-protocol service to be registered under only one id. This will also need to a way to know which codec is used when applying settings.
- Name can be changed without consequences.

# Drawbacks

# Additional Information