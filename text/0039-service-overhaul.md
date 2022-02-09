***This RFC is dependent on [RFC 45](https://github.com/tytan652/rfcs/blob/protocol_api/text/0045-protocols-in-outputs-api.md).***

# Summary

- Make OBS able to accept third-party service plugins
- Register services with a unique id rather than a common one
- A service can be provided with multiple protocols
- Separate Twitch, Restream and YouTube integration and make them plugins

# Motivation

Actually even if OBS has the service API, developer can't create third-party service plugin because there is no mechanism to list those and show their own property view.

Before in OBS, in the Stream settings page, there was a combobox with two choice:
- Streaming Services
- Custom Streaming Server

Each one had his own property view because those two are the two actual registered official services.
This behavior could enable third-party service plugin to exist.

Nowadays in OBS, this page show the list of services with many new elements showed of hidden depending of the selection (like recommended settings). With also Twitch and Restream OAuth integrations. And no use of the property views provided by `rtmp-services`.

We need to restore the usage of property views and the possibility to create and use service plugin.

It will also provide the ability for some stream services to be able to made their own plugin.

*Since using the service API was not possible with the actual OBS, any breaking change in it (if there are) will not impact any plugin for OBS.*

# Design

## UI
The Stream settings page and the auto-wizard will only contain a combo box listing registered services with also the "Show All" option. And the property view of the service will be shown under it.

A "common" service is a service that is shown in the default service list. To keep this attributes with the UI, a flag will be added to Service API to allow services to not be in this list, so by default third-party plugins will be shown without the need to click on the "Show all" option.

Note: every service that are not common will have the flag added in first-party plugins so they will not be shown by default.

- Maximums are limits and one of those 3 choices:
  1. No option at all to bypass it
  2. Add a secret option that shall be added in `basic.ini` to bypass it
  3. Add a checkbox in the Advanced tab to bypass it
- Recommended settings will not provide bitrates and will be ignore-able through a checkbox in the service property.

Supported resolutions should be enforced by default and can not be disabled.

Service with an VOD/archive track feature will also have a flag. So this option will only be shown if the combo service-protocol is compatible with this feature.
This feature should be considered different from a multi-track feature.

*Personal note: Some thought about also adding a flag for bandwidth test mode are also on my mind.*

Many elements will be moved inside properties views like:
- Stream key field
- Username and password fields
- OAuth connect disconnect button
- Text with clickable link
- Maximum and recommended settings information
- Ignore "recommended setting" checkbox
- "Get Stream Key" button
- "More Info" button

## Service plugins
### Issue
#### `rtmp-services`
This plugin provide services with only one protocol, so it needs to "register" service like YouTube multiple times (HLS & RTMPS).

And also OBS is too heavily dependent of it.

#### Service integrations
Those integrations are added and use `rtmp-services` in a hacky way and not as plugins.

### Idea for new services plugins
#### `rtmp-services`
This plugin will be put in a state of deprecation. And be replaced and finally removed after one major release cycle.

Since OBS is heavily dependent of this plugin, many changes will be needed to remove this dependency.

##### Conversion `rtmp_common` and `rtmp_custom` to new services
A way to convert user service config to a new one will be added. More detail below.

#### Service integrations

Twitch, YouTube and Restream have their own integration in OBS with OAuth. But those need to be isolated as plugins (frontend if needed).

Those will plugins will provide the basic and OAuth version of the services as one services.

If OBS is built without client ids and hashes, those plugins will be built without their integration.

##### Conversion to new plugins
Everything related to auth in `basic.ini` will moved elsewhere. More detail below.

##### About dock additions (idea/draft)
A new function in the Service API to load UI additions when starting OBS or applying setting. An unload may be needed or added to the destroy function.

##### About "Manage Broadcast" in YouTube integration About dock additions (idea/draft)
YouTube adds a "broadcast manager", some services could also have the same need (e.g. PeerTube) so the button will be kept and a new function and a flag to the service API allowing this behavior will be added.

#### `custom-service`
This plugin is meant to provide a replacement for `rtmp_custom` type.

If OBS need to be heavily dependent to one plugin, it shall be this one.

The protocol will be detected (or maybe forced set by the user).
Maybe properties will show depending of the protocol.

#### `obs-services`
This plugin is meant to provide a replacements for `rtmp_common` type for services who doesn't have custom behavior or integration.

This plugin will need a brand new services.json with new format, to register each streaming service with their own id. No more things like `rtmp-common` id.
 
Those services will be able to provide multiple protocols so no more "Service - HLS" and "Service - FTL".

If a certain protocol is not available, the plugin will not register the service if the service doesn't use another protocol.
If it does, the missing protocol will not be shown. [Thanks to RFC 45](https://github.com/tytan652/rfcs/blob/protocol_api/text/0045-protocols-in-outputs-api.md).

Those services should have no specific behavior like ingest management.

But can list with which codec they are compatible if needed. [Thanks to RFC 45](https://github.com/tytan652/rfcs/blob/protocol_api/text/0045-protocols-in-outputs-api.md)

In the end, adding streaming service will only be a JSON addition for this plugin.
Only improvements will be accepted in the code of this plugin.

#### A plugin per special case
This/Those plugin(s) is/are meant to provide replacements for `rtmp_common` type for services which need a custom behavior.

This/Those plugin(s) is/are meant to have implemented and register services which need a specific behavior like custom ingest with a unique id for each one.

#### Note about the plugins
- Like this if the service needs a specific behavior, you can 'transfer' it from `obs-services` to a new plugin and keep the same id. The change will be seamless for the user.

### Conversion of services
**Downgrade will break service configuration**
A JSON or a harcoded list with old service name linked to their new id, to make OBS able to convert the `service.json` to a new one.

Auth integration config inside `basic.ini` will be also transfered to the new file.

## Other changes

### Service API changes
- Add missing get_properties2 and get_default2 functions for service

### Property API addition
- Add a property to be able to show information with text label

Since the service API is in a way unusable by third-party, any breakage will only impact OBS built-in service plugins ( so only `rtmp-services`).

### Service integrations new flow

#### How docks and CEF widget for OAuth will be created
The Front-end API needs to enable the possibility to access some `obs-browser` related feature like adding browser docks and generating widgets.

Note: Free functions for each structure that requires it because of a "dynamic" will be also added.

##### Check availability
- `bool obs_frontend_browser_available()` will indicate if OBS Studio have the `obs-browser` included and available with a Wayland check on Linux/FreeBSD. This will allow plugins to know if they can use browser features.
- `bool obs_frontend_browser_initialised()` will indicate if the CEF was initialised when the function is called.
##### Browser Widget
Internal OAuth (like Twitch and Restream) require that the Front-end API allow to add a browser widget to show the OAuth login page.

This structure will be used to send everything needed to create this widget through the API.
```C++
struct obs_frontend_browser_widget {
	/* takes QLayout */
	void *layout;
	const char *url;
	bool enable_cookie;
	DARRAY(struct obs_frontend_browser_connect) connection;
};
```
- `void *layout` will contain a `QLayout` pointer which will receive the browser widget.
- `const char *url` will contain the URL af the widget.
- `bool enable_cookie`, if true `panel_cookie` will be set on the widget rather than a `nullptr`. This will allow to keep cookie which is required.
- `DARRAY(struct obs_frontend_browser_connect) connection` is a array that will contain signal-slot connections that will be created on OBS Studio side. Required to allow url changed detection.

```C++
struct obs_frontend_browser_connect {
	const char *signal;
	/* takes QObject */
	void *signal_receiver;
	const char *slot;
};
```

`obs_frontend_add_browser_widget(struct obs_frontend_browser_widget *params)` is the function that will add a browser widget to the QLayout put in the structure.

##### Browser dock
Twitch and Restream adds browser docks, so the Front-end API needs to allow this with many parameters.

This structure will be used to send everything needed to create those docks through the API.
```C++
struct obs_frontend_browser_dock {
	const char *id;
	const char *title;
	const char *url;
	int width;
	int height;
	int min_width;
	int min_height;
	bool enable_cookie;
	struct dstr startup_script;
	DARRAY(char *) force_popup_url;
};
```
- `const char *id`, ID of the dock.
- `const char *title`, title of the dock.
- `const char *url`, URL of the dock.
- Dock default and minimum dimensions, every parameter under 80 is not used because 80 is the minimum set by OBS Studio.
- `bool enable_cookie`, if true `panel_cookie` will be set on the dock rather than a `nullptr`. This will allow to keep cookie which is required.
- `struct dstr startup_script` allow to set a startup script for the dock.
- `DARRAY(char *) force_popup_url` allow to set a list of URL to force those to popup.

`void * obs_frontend_add_browser_dock(struct obs_frontend_browser_dock *params)` is the function that will add those browser docks. It returns a `QDockWidget`. For now those docks or not shown by default.

`void obs_frontend_remove_browser_dock(void *dock)` is the function meant to remove a browser dock. Need when disconnecting from a service. It takes QDockWidget, calls delete on dock and its corresponding QAction.

##### Other functions
`void obs_frontend_delete_browser_cookie(const char *url)` will remove cookies related to the given URL.

#### How docks will be added

`OBS_FRONTEND_EVENT_FINISHED_LOADING` can be used when OBS startup to add docks.
`OBS_FRONTEND_EVENT_EXIT` to unload docks that need to be removed before exit.
`OBS_FRONTEND_EVENT_PROFILE_CHANGING` to unload dock before profile got changed.
`OBS_FRONTEND_EVENT_PROFILE_CHANGED` to load integration from the new loaded profile.

Three new front-end event will be added:
- `OBS_FRONTEND_EVENT_SERVICE_CHANGING` emitted before the service got changed (new id) like when a user switches between services. 
- `OBS_FRONTEND_EVENT_SERVICE_CHANGED` emitted when the service was changed (new id) like when a user switches between services.
- `OBS_FRONTEND_EVENT_SERVICE_UPDATING` emitted before the service got a parameter updated (no id change) like when a user has changed one of the available properties/settings of the actual service.
- `OBS_FRONTEND_EVENT_SERVICE_UPDATED` emitted when the service had a parameter updated (no id change) like when a user has changed one of the available properties/settings of the actual service.

Each service integration plugin will react to at least the two first to add or remove his related docks.

#### How docks will keep their state between session and profile
Actually the dock state is global in the config, but with docks that appear depending on the service which is per profile which is still the case with "old" integrations.

This which can lead to dock state losses between profile switching and exiting. The "old" integration actually store a dock state in the profile config and restore it after being the integration loaded is loaded.

But making restoring a dock state from a plugin should be the last resort.
Making it per profile could avoid this.

Integrations docks will be added after specific frontend event.
So after each of them the dock state needs to be restored.

`OBS_FRONTEND_EVENT_FINISHING_LOADING` (finishing not finished) needs to be created
to allow adding docks and restore dock state before `OBS_FRONTEND_EVENT_FINISHED_LOADING` is emitted.

Like this `OBS_FRONTEND_EVENT_FINISHED_LOADING` keep his meaning and does not get a restore dock state after it.

#### Manage Broadcast
YouTube is not the only service that could have or need a "Manage Broadcast" button. So adding a way to make other service plugin able to use it.

- Add a unique callback meant to be called when the "Manage Broadcast" button is clicked.
- Add a callback setter in the frontend-api which will make the button show up. This setter will always override the previous callback.
- Add a callback unsetter in the frontend-api which will make the button hide. This unsetter will have the callback as a parameter to check if the set callback should really be unset.

### Service JSON format

#### Old version

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

##### Issues with this format
- About `"output"`, OBS consider every service as RTMP if not added and it's not a recommended settings at all. It makes OBS use the right protocol for the service. Also this prevent a service of being multi-protocol.
- About `"recommended"`, most of the options seems to H264 related
- `"common"`, the name makes it not understandable at the first sight maybe adding some documention would be good thing.

#### New format (conversion in JSON Schema with changes is in WIP)
The new format will be parsable with a new interface library named `service-json-parser` to allow first-party plugins to read this format rather than recreating the wheel.

The new format can be considered as the version 4.

This format will be defined by JSON Schemas (Draft 2020-12):
- `commonPattern.schema.json`: regex pattern use multiple times
- `protocolDefs.schema.json`: regex pattern and enum about protocols to ease protocol additions or removals in schemas
- `service.schema.json`: define the service object itself (*still in WIP*)
- `obs-services.schema.json`: `obs-services` JSON Schema

Those are all present in `0039-json-schemas` folder.

*Documention in schemas are in progress, and a schema for integration plugin should also be made.*

JSON Schema allows to validate `services.json` (in `obs-services` case) when a PR is made against it. *Draft python script already made.*

The use of JSON Schema allows to create specific additions for a specific plugin by creating a unique schema for it.

<!-- Multi-service plugins should have a JSON root that look like this:
```json
{
    "format_version": 4,
    "services": []
}
```
`"services"` is an array of service objects.

Service plugin like integrations one should have a JSON root that look like this:
```json
{
    "format_version": 4,
    "service": {}
}
```
`"service"` is the service object itself. -->

<!-- ##### Plugins specific additions
Those additions are added like extensions to the service object format.

Getters for those customisations are implemented in the plugin itself.
- In `obs-services`:
  - `"id"` (required): service identifier, services are no longer identified by their name. Added because this plugin is multi-services.
  - `"name"` (required): name of the service. Added because this plugin is multi-services.
  - `"common"` (optional, false by default): this object is added in service objects of this plugin to identify which service in "common" or not. No third-party plugin should have this behavior.
- In `obs-youtube`:
  - Servers name are translated.
  - `"name_suffix"` in `"servers"` (optional): suffix added to the server name after being translated. In this case "(legacy RTMP)" is the suffix added to translated server names to avoid creating more than 2 translation entries.

Specific documentations about adding service in `obs-services` should be made.

##### Service object format
<sup>S</sup>: Should be required by any first-party plugin

<sup>O</sup>: Optional in `obs-services`, can be required or not used at all in first-party one-service plugins depending on their implementations.

- `"more_info_link"`<sup>O</sup>: link with more info about the service.
- `"stream_key_link"`<sup>O</sup>: link where the user can find his stream key.
- `"servers"`<sup>S</sup>: array of servers.
  - `"name"` (required): name of the server.
  - `"protocol"` (required if if the url prefix can't help identify the protocol, like HLS protocol): protocol identifier (e.g. RTMP, SRT, RIST).
  - `"url"` (required): url of the server.
- `"supported_codecs"`<sup>O</sup>: codec not supported by a protocol will not be used. If not set, fallback to what the protocol can support.
  - `"video"`<sup>O</sup>: video codecs supported by the service.
    - `"any_protocol"`<sup>O</sup>: array of strings with codecs' name supported by the service for any protocol.
    - *`"protocol name"`*<sup>O</sup>: (e.g. `"HLS":`) array of strings with codecs' name supported by only this protocol. This array is merged with `"any_protocol"`.
  - `"audio"`: audio codecs supported by the service.
    - `"any_protocol"`<sup>O</sup>: array of strings with codecs' name supported by the service for any protocol.
    - *`"protocol name"`*<sup>O</sup>: (e.g. `"HLS":`) array of strings with codecs' name only supported by this protocol. This array is merged with `"any_protocol"`.
- `"supported_resolutions"`<sup>O</sup>: array of strings with resolutions (e.g. `"1280x720"`) supported by the service for any protocol.
  - `"with_framerates"`<sup>O</sup>: default to false. If false each entry shall be put as this "1920x1080", if true it will be put with a framerate as this "1920x1080@30". Only those framerate will be be considered supported.
  - `"array"`(required): Array of resolutions
- `"maximums"`<sup>O</sup>: maximums allowed by the service.
  - `"fps"`<sup>O</sup>: maximum framerate allowed by the service. May be overrided if `"supported_resolutions"` is set with framerates (if 1080p is limited to 30 FPS and 720p to 60, the maximum will change depending on the resolution).
  - `"video_bitrate"`<sup>O</sup>: maximum video bitrate. Can be set per codec. Unused if `"video_bitrate_matrix"` is set.
    - `"any_codec"`<sup>O</sup>: bitrate for any codec.
    - *`"codec name"`*<sup>O</sup> (requires to be put in `"supported_codecs"` firstly): (e.g. `"av1"`) bitrate for this codec. It overrides `"any_codec"`.
  - `"video_bitrate_matrix"`<sup>O</sup> (requires `"supported_resolutions"` with framerates): maximum bitrate based on resolutions and framerates from `"supported_resolutions"`.
    - *`"CX x CY @ FPS"`*<sup>O</sup> (**one per resolution**): (e.g. `"1280x720@60":`) maximum bitrate for supported resolution of this service. Can be set per codec.
      - `"any_codec"`<sup>O</sup>: bitrate for any codec.
      - *`"codec name"`*<sup>O</sup> (requires to be put in `"supported_codecs"` firstly): (e.g. `"av1":`) bitrate for this codec. It overrides `"any_codec"`.
  - `"audio_bitrate"`<sup>O</sup>: maximum audio bitrate. Can be set per codec.
    - `"any_codec"`<sup>O</sup>: bitrate for any codecs.
    - *`"codec name"`*<sup>O</sup> (requires to be put in `"supported_codecs"` firstly): (e.g. `"av1":`) bitrate for this codec. It overrides `"any_codec"`.
- `"recommended"`<sup>O</sup>: recommended settings that become the defaults when using this service.
  - `"keyint"`<sup>O</sup>: is now per video codec.
    - `"any_codec"`<sup>O</sup>: keyframe interval for any codec.
    - *`"codec name"`*<sup>O</sup> (requires to be put in `"supported_codecs"` firstly): (e.g. `"av1"`) keyframe interval for this codec. It overrides `"any_codec"`.
  - `"bframes"`<sup>O</sup>: is now per video codec.
    - `"any_codec"`<sup>O</sup>: b-frames for any codec.
    - *`"codec name"`*<sup>O</sup> (requires to be put in `"supported_codecs"` firstly): (e.g. `"av1"`) b-frames for this codec. It overrides `"any_codec"`.
  - `"profile"`<sup>O</sup>: is now per video codec.
    - No `"any_codec"` because profile is surely different between codecs.
    - *`"codec name"`*<sup>O</sup> (requires to be put in `"supported_codecs"` firstly): (e.g. `"av1"`) profile for this codec if available.  It overrides `"any_codec"`.
  - `"x264opts"`<sup>O</sup>: no change.
  - `"fps"`<sup>O</sup>: recommended framerate for this service. Unused if `"supported_resolutions"` is set with framerates.
  - `"video_bitrate"`<sup>O</sup>: recommended video bitrate. Can be set per codec. Unused if `"video_bitrate_matrix"` is set.
    - `"any_codec"`<sup>O</sup>: bitrate for any codec.
    - *`"codec name"`*<sup>O</sup> (requires to be put in `"supported_codecs"` firstly): (e.g. `"av1"`) bitrate for this codec. It overrides `"any_codec"`.
  - `"video_bitrate_matrix"`<sup>O</sup> (requires `"supported_resolutions"` with with framerates): recommended bitrate based on resolutions and framerates from `"supported_resolutions"`.
    - *`"CX x CY @ FPS"`*<sup>O</sup> (**one per resolution**): (e.g. `"1280x720@60":`) recommended bitrate for supported resolution of this service.
      - `"any_codec"`<sup>O</sup>: bitrate for any codec.
      - *`"codec name"`*<sup>O</sup> (requires to be put in `"supported_codecs"` firstly): (e.g. `"av1":`) for this codec. It overrides `"any_codec"`.
  - `"audio_bitrate"`<sup>O</sup>: maximum audio bitrate. Can be set per codec.
    - `"any_codec"`<sup>O</sup>: bitrate for any codecs.
    - *`"codec name"`*<sup>O</sup> (requires to be put in `"supported_codecs"` firstly): (e.g. `"av1":`) bitrate for this codec. It overrides `"any_codec"`.-->

##### Example
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

##### Advantages of this format

- Recommended settings are really recommended settings.
- Recommendation and maximums are two separated thing.
- Some settings are now per protocol or per codec, this allow multi-protocol service to be registered under only one id. This will also need to a way to know which codec is used when applying settings.
- Name can be changed without consequences.

##### Possible evolution
Add to the service object an url that lead to an updated service object hosted by the stream service itself.

### Service ID naming scheme
- Only lower case letter
- Hyphen `-` are only used to distinguish two service providing the same stream service but with noteworthy differences like NicoNico (free & premium).

### Things to be reworked
- Verification with video and output settings to see (if not ignored) if the settings for the service are respected.

### About codecs
Some services could support a protocol without supporting all the compatible codecs, so adding a new field like `supported_codecs` for audio and video may be needed.

# Drawbacks
Dockstate from integration are not recoverable.

# Additional Information
