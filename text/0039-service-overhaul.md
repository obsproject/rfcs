# Summary

- Make OBS able to accept third-party service plugins
- Register services with a unique id rather than a common one
- A service can be provided with multiple protocols
- Separate Twitch, Restream and YouTube integration and make them plugins

# Motivation

Actually even if OBS has the Service API, developer can't create third-party service plugin because there is no mechanism to use them at all.

Before OBS in the Stream settings page, property views were used for services but with only two registered services `rtmp_common` which contain all services and `rtmp_custom` for custom servers.

Nowadays in OBS, this page show the list of services with many new elements showed of hidden depending of the selection (like recommended settings). With also Twitch and Restream OAuth integrations. And no use of the property views provided by `rtmp-services`.

This need to be refactored to re-introduce property views for the service and for the protocol output.

This will also provide the ability for some stream services to be able to made their own plugin.

# Design

## Settings UI

Many elements of the settings UI related to services will be moved inside services properties views like:
- Stream key field
- Username and password fields
- OAuth connect disconnect button
- Text with clickable link
- Maximum and recommended settings information
- Ignore "recommended setting" checkbox
- "Get Stream Key" button
- "More Info" button

This streaming output will be selected through the settings windows and no longer auto-selected while starting the stream and its property view will be placed below the service properties view.

The service output settings (if any) will be saved in the `service.json` file.

Advanced network settings will also be replace by the output properties view below the service properties view, since those are meant for `"rtmp_output"` (RTMP(S) output).

**TODO/REDO: All services must at least be compatible with H264 because of how simple output mode is designed.**

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
- `obs_frontend_set_streaming_service()` because the output selection is done in the settings windows and is no longer done while the stream is starting will now swap the output if the service protocol and the actual output does not match.
- `obs_frontend_save_streaming_service()` will save service output settings.

## About the FTL protocol

The FTL protocol is actually planned to be deprecated once a new protocol is added after this RFC is implemented.

Any services that only support FTL will follow the deprecation cycle and be dropped at the same time as FTL if those services did not added support for another protocol.

## Service plugins
### Service API

Adding to `obs_service_info`:
  - `uint32_t flags` with the following flags:
    - `OBS_SERVICE_DEPRECATED`: (**TODO: Re-think about it because of internal**) The service is deprecated and will not be shown in the UI combobox list.
    - `OBS_SERVICE_INTERNAL`: The service is meant to be used internally in some plugin (e.g., WebSocket, Scripting) and usually not directly exposed in the UI.
    - `OBS_SERVICE_UNCOMMON`: The service can be hidden behind a "Show All/More" option UI/UX-wise.
    - `OBS_SERVICE_ANY_PROTOCOL`: The service supports any protocol, mainly meant for `"custom_service"`
  - `const char *supported_protocols`: Protocol supported by the service, required if `OBS_SERVICE_ANY_PROTOCOL` is not set as a flag.
  - `obs_properties_t *(*get_properties2)(void *data, void *type_data)`: Same as its non-2 variant but give access to the `type_data` pointer.

**TODO: Adding `get_default2()` functions for service might be required.**

### `rtmp-services`

Services provided by this plugin (`"rtmp_custom"`, `"rtmp_common"`) will be deprecated and completely unused in OBS Studio.

Those are deprecated rather than completely removed to allow scripting and plugins to migrate if they happen to use them.

Its service JSON will no longer be updated.


### `custom-services`

If OBS Studio need to be heavily dependent to one service plugin, it must be this one.

This plugin is meant to provide replacements for the `"rtmp_custom"` type.

#### `"custom_service"`

This the only service of this plugin thar will be exposed to the user through the UI.

The protocol will be selected by the user and the properties view will change depending of the protocol.

**TODO: Add about third-party support in `"custom_service"` type**

#### Custom type per protocol

Service types that support only the protocol inside their id (`"custom_`*insert lowercase protocol acronym*`"`) for any first-party protocol except FTL (deprecation).

Services that will only support the first-party protocol related to their id/type.
The flag `OBS_SERVICE_INTERNAL` will be applied to not show them in the UI list. Those are meant for being used in WebSocket and scripting.

### `obs-services`

This plugin is meant to provide a replacements for `"rtmp_common"` type for services who doesn't have custom behavior or integration.

This plugin will need a brand new services.json with new format, to register each streaming service with their own id. No more things like `"rtmp_common"` id.
 
Those services will be able to provide multiple protocols so no more "Service - HLS" and "Service - FTL".

If a certain protocol is not available, the UI will not show it if the service doesn't support another protocol. Same goes for codecs.
If it does, the missing protocol will not be shown.

Those services should have no specific behavior like ingest management.

In the end, adding streaming service will only be a JSON addition for this plugin.
Only improvements will be accepted in the code of this plugin.

#### Service ID naming scheme
- Only lower case letter
- (**Might be removed**) All prefixed by `obs-` in code, to avoid potential id conflict with streaming service third-party plugins.
- Hyphen `-` are only used to distinguish two service providing the same stream service but with noteworthy differences like NicoNico (free & premium).
- Like this if the service needs a specific behavior, you can 'transfer' it from `obs-services` to a new first-party plugin and keep the same id. The change will be seamless for the user if done properly.

### Special case plugins

Those plugins are meant to provide replacements for `"rtmp_common"` type for services that were working around the JSON usually by adding ingest management code and is not a service with an integration.

So those services that match those will have their own plugin.

4 services are concerned:
- Dacast
- Nimo TV
- SHOWROOM
- YouNow (FTL only, also falls under the FTL deprecation cycle)

Beside the `"rtmp_common"` part being replace by a unique service type, most of the ingest code will stay identical.

The maintenance of the code of those services after implementation will be on the organisation that originaly added them to `rtmp-services`.

### Conversion (WIP Needs a re-work)

**Downgrade will break service configuration**
A JSON or a harcoded list with old service name linked to their new id, to make OBS able to convert the `service.json` to a new one.

Auth integration config inside `basic.ini` will be also transfered under a new section if in service settings.

### Integrations

Twitch, YouTube and Restream have their own integration in OBS with OAuth. But those need to be isolated as plugins.

Those will plugins will provide the basic and OAuth version of the services as one services.

If OBS is built without client ids and hashes, those plugins will be built without their integration.

#### Dock state

Actually the dock state is global in the config, but with docks that appear depending on the service which is per profile which is still "also" the case with "old" integrations.

This can lead to dock state losses between profile switching and exiting. The "old" integration actually store a dock state in the profile config and restore it after the integration is is loaded.

But making restoring a dock state from a plugin should should not be considered at all.
Making it per profile could avoid this.

Integrations docks will be added after specific frontend event.
So after each of them the dock state needs to be restored.

- `OBS_FRONTEND_EVENT_FINISHING_LOADING` (finish**ing** not finished) needs to be created to allow adding docks. And restore dock state before if docks are indeed added before `OBS_FRONTEND_EVENT_FINISHED_LOADING` is emitted.

- `OBS_FRONTEND_EVENT_PROFILE_CHANGED`, profile dock state will be loaded after a profile is changes.

Also they will be removed after specific frontend event.
So before each of them the dock state needs to be saved.

- `OBS_FRONTEND_EVENT_PROFILE_CHANGING`, profile dock state will be saved before a profile is changes.

- `OBS_FRONTEND_EVENT_EXIT`, profile dock state will be saved before while OBS is exiting.

#### Dock addition (**The concept needs to be tested**)

Integration plugins will need events/signals to be able to add and remove their docks at the right moment.


3 signals will be added to Core OBS signaling:

- "service_create (ptr service)"
- "service_update (ptr service)"
- "service_destroy (ptr service)"

A variant of `obs_service_create()` to spawn signal-less (and maybe even private) service will be needed to allow protocol checking in settings.

The plugin will have to store if `OBS_FRONTEND_EVENT_FINISHED_LOADING` was passed or not.

If created before `OBS_FRONTEND_EVENT_FINISHED_LOADING`, a front-event callback will be added to react to `OBS_FRONTEND_EVENT_FINISHING_LOADING` to add docks (if integration connected) and then removed itself.

When created/updated (after `OBS_FRONTEND_EVENT_FINISHED_LOADING`) the service will add the docks (if integration connected).

When `OBS_FRONTEND_EVENT_PROFILE_CHANGING` `OBS_FRONTEND_EVENT_EXIT` or the service is destroyed, docks will be removed (if integration connected).

#### Browser features
The Front-end API needs to enable the possibility to access some `obs-browser` related feature like adding browser docks and generating widgets.

Note: Free functions for each structure that requires it because of a "dynamic" type will be also added.

`bool obs_frontend_browser_available()` will indicate if OBS Studio have the `obs-browser` included and available with a Wayland check on Linux/FreeBSD. This will allow plugins to know if they can use browser features.

CEF should be initialised after `OBS_FRONTEND_EVENT_FINISHED_LOADING`.

```C++
struct obs_frontend_browser_connect {
	void *qobject; // QObject
	const char *slot; // SLOT()
};

struct obs_frontend_browser_params {
	const char *url;
	bool enable_cookie;
	struct dstr startup_script;
	DARRAY(char *) force_popup_url;
        DARRAY(struct obs_frontend_browser_connect) title_changed;
        DARRAY(struct obs_frontend_browser_connect) url_changed;
};
```

- `const char *url`, URL of the dock.
- `bool enable_cookie`, if true `panel_cookie` will be set on the dock rather than a `nullptr`. This will allow to keep cookie which is required.
- `struct dstr startup_script` allow to set a startup script for the dock.
- `DARRAY(char *) force_popup_url` allow to set a list of URL to force those to popup.
- `DARRAY(struct obs_frontend_browser_connect) title_changed` allow to connect QCefWidget `titleChanged()` signal to a slot
- `DARRAY(struct obs_frontend_browser_connect) url_changed` allow to connect QCefWidget `urlChanged()` signal to a slot

`bool obs_frontend_add_browser_dock(const char *id, const char *title, struct obs_frontend_browser_dock *params)` is the function that will add those browser docks to the UI.

`void *obs_frontend_create_browser_widget(struct obs_frontend_browser_params *params)` is the function that will return a browser widget (QCefWidget) that we can cast to a QWidget for OAuth or a custom chat dock (e.g. Youtube).

`void obs_frontend_delete_browser_cookie(const char *url)` will remove cookies related to the given URL.

#### Broadcast flow (WIP)
YouTube is not the only service that could have or need a "Manage Broadcast" button. So adding a way to make other service plugin able to use it.

- Add a callback bond to a service pointer meant to be called when the "Manage Broadcast" button is clicked.
- Add a callback setter in the frontend-api which will make the button show up.
- Add a callback unsetter in the frontend-api which disable the button.


#### Twitch VOD track
This will be the **only one** custom feature that will be allowed in OBS Studio UI code.

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
- About `"recommended"`, most of the options seems to H264 related
- `"common"`, the name makes it not understandable at the first sight maybe adding some documention would be good thing.

### New format (WIP)

The new format will rely on more strict JSON schemas.

Those JSON Schemas (Draft 2020-12) will be:
- `protocolDefs.json`: enums, regex patterns and properties related to protocols to ease protocol additions or removals in schemas
- `codecDefs.json`: enums, regex patterns and properties related to codecs to ease codec additions or removals in schemas
- `service.json`: define the service object itself, will be used for every service plugin htat needs a JSON
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