# Summary

- Make OBS able to accept third-party service plugins
- Register services with a unique id rather than a common one
- A service can be provided with multiple protocols
- Stricter application of maximums and format compatibility (rate control, color format/space/range)
- Separate Twitch, Restream and YouTube integration and make them plugins

# Motivation

Actually even if OBS has the Service API, developer can't create third-party service plugin because there is no mechanism to use them at all.

Before OBS in the Stream settings page, property views were used for services but with only two registered serices `rtmp-common` which containe all services and `rtmp-custom` for custom servers.

Nowadays in OBS, this page show the list of services with many new elements showed of hidden depending of the selection (like recommended settings). With also Twitch and Restream OAuth integrations. And no use of the property views provided by `rtmp-services`.

This need to be refactored to re-introduce property for the service and for the protocol output.

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

Advanced network settings will also be replace by protocol output properties view below the service properties view.

### Common, uncommon

Spoiler: Streaming services will be individually registered, so no more `rtmp-common` that represent various services.

So to hide services behind the "Show All" option, a flag (`OBS_SERVICE_UNCOMMON`) will be added to the Service API to indicate that the service is "uncommon".

So third-party services are shown in the list by default.

### Maximums and recommended

Maximums and supported resolutions will be strict (e.g. Twitch allows 6000 kbps, more will not be accepted), the UI will not allow to save incompatible settings.

TODO: Think about recommendation

### Audio track(s)

The service needs to be able to report how it support audio tracks:
- Only one track
- Two with a archve track (VOD Track)
- Multi-track

TODO IDEA: Enum with one track, archive track (VOD track) and multi-track (only custom service) since protocol makes usage of flags complicated.

### Rate control

Services by default will ask for only CBR, some service may want something else or allow various.

Both service and encoder will need to be able to expose this to allow the UI to check for compatibility.

The UI will not allow to save incompatible settings.

### Color format/space/range

Aside from RTMP(S) (and FTL), newer protocol allow codecs that support 10-bit format and HDR.

The UI will need to not allow service that do not work with selected settings.

## Service plugins
### Service API

Add missing get_properties2 and get_default2 functions for service

### `custom-service`

This plugin is meant to provide a replacement for `rtmp_custom` type.

If OBS need to be heavily dependent to one plugin, it shall be this one.

The protocol will be detected (or maybe forced set by the user, e.g. HLS).
Properties will change depending of the protocol.

### `obs-services`

This plugin is meant to provide a replacements for `rtmp_common` type for services who doesn't have custom behavior or integration.

This plugin will need a brand new services.json with new format, to register each streaming service with their own id. No more things like `rtmp-common` id.
 
Those services will be able to provide multiple protocols so no more "Service - HLS" and "Service - FTL".

If a certain protocol is not available, the UI will not show it if the service doesn't support another protocol.
If it does, the missing protocol will not be shown.

Those services should have no specific behavior like ingest management.

In the end, adding streaming service will only be a JSON addition for this plugin.
Only improvements will be accepted in the code of this plugin.

#### Service ID naming scheme
- Only lower case letter
- All prefixed by `obs-` in code, to avoid potential id conflict with streaming service third-party plugins.
- Hyphen `-` are only used to distinguish two service providing the same stream service but with noteworthy differences like NicoNico (free & premium).
- Like this if the service needs a specific behavior, you can 'transfer' it from `obs-services` to a new first-party plugin and keep the same id. The change will be seamless for the user.

### Special case plugins

This/Those plugin(s) is/are meant to provide replacements for `rtmp_common` type for services which need a custom behavior.

This/Those plugin(s) is/are meant to have implemented and register services which need a specific behavior like custom ingest with a unique id for each one.

### Conversion (WIP Needs a re-work)

**Downgrade will break service configuration**
A JSON or a harcoded list with old service name linked to their new id, to make OBS able to convert the `service.json` to a new one.

Auth integration config inside `basic.ini` will be also transfered to the new file.

### Integrations

Twitch, YouTube and Restream have their own integration in OBS with OAuth. But those need to be isolated as plugins.

Those will plugins will provide the basic and OAuth version of the services as one services.

If OBS is built without client ids and hashes, those plugins will be built without their integration.

#### Dock state

Actually the dock state is global in the config, but with docks that appear depending on the service which is per profile which is still "also" the case with "old" integrations.

This which can lead to dock state losses between profile switching and exiting. The "old" integration actually store a dock state in the profile config and restore it after being the integration loaded is loaded.

But making restoring a dock state from a plugin should be the last resort.
Making it per profile could avoid this.

Integrations docks will be added after specific frontend event.
So after each of them the dock state needs to be restored.

`OBS_FRONTEND_EVENT_FINISHING_LOADING` (finishing not finished) needs to be created
to allow adding docks and restore dock state before `OBS_FRONTEND_EVENT_FINISHED_LOADING` is emitted.

Like this `OBS_FRONTEND_EVENT_FINISHED_LOADING` keep his meaning and does not get a restore dock state after it.

#### Dock addition

Integration plugins will need events to be able to add and remove service at the right moment.

`OBS_FRONTEND_EVENT_FINISHING_LOADING` can be used when OBS startup to add docks.
`OBS_FRONTEND_EVENT_EXIT` to unload docks that need to be removed before exit.
`OBS_FRONTEND_EVENT_PROFILE_CHANGING` to unload dock before profile got changed.
`OBS_FRONTEND_EVENT_PROFILE_CHANGED` to load integration from the new loaded profile.

Three new front-end event will be added (**Needs re-work for future multi-stream RFC**):
- `OBS_FRONTEND_EVENT_SERVICE_CHANGING` emitted before the service got changed (new id) like when a user switches between services. 
- `OBS_FRONTEND_EVENT_SERVICE_CHANGED` emitted when the service was changed (new id) like when a user switches between services.
- `OBS_FRONTEND_EVENT_SERVICE_UPDATING` emitted before the service got a parameter updated (no id change) like when a user has changed one of the available properties/settings of the actual service.
- `OBS_FRONTEND_EVENT_SERVICE_UPDATED` emitted when the service had a parameter updated (no id change) like when a user has changed one of the available properties/settings of the actual service.

Each service integration plugin will react to at least the two first to add or remove his related docks.

#### Browser features
The Front-end API needs to enable the possibility to access some `obs-browser` related feature like adding browser docks and generating widgets.

Note: Free functions for each structure that requires it because of a "dynamic" will be also added.

`bool obs_frontend_browser_available()` will indicate if OBS Studio have the `obs-browser` included and available with a Wayland check on Linux/FreeBSD. This will allow plugins to know if they can use browser features.
`bool obs_frontend_browser_initialised()` will indicate if the CEF was initialised when the function is called required for OAuth through CEF.

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

`bool obs_frontend_add_browser_dock(const char *id, struct obs_frontend_browser_dock *params)` is the function that will add those browser docks to the UI.

`void *obs_frontend_create_browser_widget(struct obs_frontend_browser_params *params)` is the function that will return a browser widget (QCefWidget) that we can cast to a QWidget for OAuth or a custom chat dock (e.g. Youtube).

`void obs_frontend_delete_browser_cookie(const char *url)` will remove cookies related to the given URL.

#### Broadcast flow (WIP)
YouTube is not the only service that could have or need a "Manage Broadcast" button. So adding a way to make other service plugin able to use it.

- Add a unique callback meant to be called when the "Manage Broadcast" button is clicked.
- Add a callback setter in the frontend-api which will make the button show up. This setter will always override the previous callback.
- Add a callback unsetter in the frontend-api which will make the button hide. This unsetter will have the callback as a parameter to check if the set callback should really be unset.


## Service JSON

### Old version

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

The new format will rely on a more strict JSON schemas.

Those JSON Schemas (Draft 2020-12) will be:
- `commonPattern.schema.json`: regex pattern use multiple times
- `protocolDefs.schema.json`: regex pattern and enum about protocols to ease protocol additions or removals in schemas
- `service.schema.json`: define the service object itself (*still in WIP*)
- `obs-services.schema.json`: `obs-services` JSON Schema

Those are all present in `0039-json-schemas` folder.

*Documention in schemas are in progress, and a schema for integration plugin should also be made.*

JSON Schema allows to validate `services.json` (in `obs-services` case) when a PR is made against it.

The use of JSON Schema allows to create specific additions for a specific plugin by creating a unique schema for it.

#### Example
Here is a example with `obs-services` in mind:
```json
{
    "format_version": 4,
    "services": [
        {
            "id": "example",
            "name": "Example of stream services",
            "more_info_link": "https://example.com/more_info",
            "stream_key_link": "https://example.com/stream_key",
            "servers": [
                {
                    "name": "SRT Server",
                    "url": "srt://example.com"
                },
                {
                    "name": "RTMPS Server",
                    "url": "rtmps://example.com/server"
                },
                {
                    "name": "FTL Server",
                    "url": "ftl://example.com/"
                },
                {
                    "name": "HLS Server",
                    "protocol": "HLS",
                    "url": "https://example.com/http_upload_hls?cid={stream_key}&copy=0&file=out.m3u8"
                }
            ],
            "supported_codecs": {
                "video": {
                    "*": [
                        "h264"
                    ],
                    "SRT": [
                        "av1"
                    ]
                },
                "audio": {
                    "*": [
                        "aac"
                    ],
                    "SRT": [
                        "opus"
                    ]
                }
            },
            "supported_resolutions": [
                "1920x1080@60",
                "1920x1080@30",
                "1280x720@60",
                "1280x720@30"
            ],
            "maximums": {
                "fps": 60,
                "video_bitrate": {
                    "any_codec": 9000,
                    "av1": 8000,
                },
                "audio_bitrate": {
                    "any_codec": 320,
                    "opus": 640,
                },
                "video_bitrate_matrix": {
                    "1920x1080@60": {
                        "h264": 9000,
                        "av1": 8000
                    },
                    "1920x1080@30": {
                        "h264": 6000,
                        "av1": 5000
                    },
                    "1280x720@60": {
                        "h264": 6000,
                        "av1": 5000
                    },
                    "1280x720@30": {
                        "h264": 4000,
                        "av1": 3000
                    }
                },
            },
            "recommended": {
            }
        }
    ]
}
```

#### Advantages of this format

- Recommended settings are really recommended settings.
- Recommendation and maximums are two separated thing.
- Some settings are now per protocol or per codec, this allow multi-protocol service to be registered under only one id. This will also need to a way to know which codec is used when applying settings.
- Name can be changed without consequences.

# Drawbacks

# Additional Information