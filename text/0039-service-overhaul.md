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

A "common" service is a service that is shown that is shown in the default service list. To keep this attributes with the UI, a flag will be added to Service API to allow services to not be in this list, so by default third-party plugins will be shown without the need to click on the "Show all" option.

Service with an VOD/archive track feature will also under a flag. So this option will only be shown if the combo service-protocol is compatible with this feature.
This feature should be considered different from a multi-track feature.

Many elements will be moved inside properties views like:
- Stream key field
- Username and password fields
- OAuth connect disconnect button
- Text with clickable link
- Maximum and recommended settings label (and maybe their ignore checkboxes)
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

The notion of maximum and recommended shall be distinguished.
Maximum, supported resolutions, recommended settings will be ignorable with their respective checkboxes.

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
The front-end API will need to have a way to access and use `cef` and `panel_cookies` from the main window.

#### How docks will be added

`OBS_FRONTEND_EVENT_FINISHED_LOADING` can be used when OBS startup to enable docks.

Three new front-end event will be added:
- `OBS_FRONTEND_EVENT_SERVICE_CHANGED` emitted when the service was changed (new id).
- `OBS_FRONTEND_EVENT_SERVICE_UPDATED` emitted when the service had a parameter updated (no id change).

Each service integration plugin will react to at least the two first to add or remove his related docks.

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
- The naming scheme is a mix of object name with space and some with underscores.
- About `"output"`, OBS consider every service as RTMP if not added and it's not a recommended settings at all. It makes OBS use the right protocol for the service. Also this prevent a service of being multi-protocol.
- About `"recommended"`, most of the options seems to H264 related
- `"common"`, the name makes it not understandable at the first sight maybe adding some documention would be good thing.

#### New format (draft)
Here is a example:
```json
{
    "format_version": 1,
    "services": [
        {
            "id": "example",
            "name": "Example of stream services",
            "common": false,
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
                    "any_protocol": [
                        "h264"
                    ],
                    "SRT": [
                        "av1"
                    ]
                },
                "audio": {
                    "any_protocol": [
                        "aac"
                    ],
                    "SRT": [
                        "opus"
                    ]
                }
            },
            "supported_resolutions": {
                "with_framerate": true,
                "array": [
                    "1920x1080@60",
                    "1920x1080@30",
                    "1280x720@60",
                    "1280x720@30"
                ]
            },
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
                        "any_codec": 9000,
                        "av1": 8000
                    },
                    "1920x1080@30": {
                        "any_codec": 6000,
                        "av1": 5000
                    },
                    "1280x720@60": {
                        "any_codec": 6000,
                        "av1": 5000
                    },
                    "1280x720@30": {
                        "any_codec": 4000,
                        "av1": 3000
                    }
                },
            },
            "recommended": {
                "video": {},
                "video_bitrate_matrix": {},
                "audio": {},
            }
        }
    ]
}
```

- `"format_version"`: no change.
- `"services"`: array of services.
  - **`"id"`**: service identifier, services are no longer identified by their name.
  - `"name"` (if multiple servers): no change.
  - `"common"` (optional): no change.
  - `"more_info_link"` (optional): no change.
  - `"stream_key_link"` (optional): no change.
  - ~~`"alt_names"`~~: removed.
  - `"servers"`: array of servers.
    - `"name"`: no change.
    - **`"protocol"`** (optional): needed if the url prefix can't help identify the protocol.
    - `"url"`: no change.
  - **`"supported_codecs"`** (optional): codec not supported by a protocol will not be used. If not set, fallback to what the protocol support.
    - **`"video"`** (optional): video codecs supported by the service.
      - **`"any_protocol"`** (optional): array of strings with codecs' name supported by the service for any protocol.
      - ***`"protocol name"`*** (optional): (e.g. `"HLS":`) array of strings with codecs' name supported by only this protocol. This array is merged with `"any_protocol"`.
    - **`"audio"`** (optional): audio codecs supported by the service.
      - **`"any_protocol"`** (optional): array of strings with codecs' name supported by the service for any protocol.
      - ***`"protocol name"`*** (optional): (e.g. `"HLS":`) array of strings with codecs' name only supported by this protocol. This array is merged with `"any_protocol"`.
  - **`"supported_resolutions"`** (optional): array of strings with resolutions (e.g. `"1280x720"`) supported by the service for any protocol.
    - **`"with_framerates"`** (optional): default to false. If false each entry shall be put as this "1920x1080", if true it will be put with a framerate as this "1920x1080@30". Only those framerate will be be considered supported.
    - **`"array"`**: Array of resolutions
  - **`"maximums"`** (optional): maximums allowed by the service.
    - **`"fps"`** (optional): maximum framerate allowed by the service. May be overrided if `"supported_resolutions"` is set with framerates (if 1080p is limited to 30 FPS and 720p to 60, the maximum will change depending on the resolution).
    - **`"video_bitrate"`** (optional): maximum video bitrate. Can be set per codec. Unused if `"video_bitrate_matrix"` is set.
      - **`"any_codec"`** (optional): bitrate for any codec.
      - ***`"codec name"`*** (optional, requires to be put in `"supported_codecs"` firstly): (e.g. `"av1"`) bitrate for this codec. It overrides `"any_codec"`.
    - **`"video_bitrate_matrix"`** (optional but requires `"supported_resolutions"` with framerates): maximum bitrate based on resolutions and framerates from `"supported_resolutions"`.
      - ***`"CX x CY @ FPS"`*** (one per resolution): (e.g. `"1280x720@60":`) maximum bitrate for supported resolution of this service. Can be set per codec.
        - **`"any_codec"`** (optional): bitrate for any codec.
        - ***`"codec name"`*** (optional, requires to be put in `"supported_codecs"` firstly): (e.g. `"av1":`) bitrate for this codec. It overrides `"any_codec"`.
    - **`"audio_bitrate"`** (optional): maximum audio bitrate. Can be set per codec.
      - **`"any_codec"`** (optional): bitrate for any codecs.
      - ***`"codec name"`*** (optional, requires to be put in `"supported_codecs"` firstly): (e.g. `"av1":`) bitrate for this codec. It overrides `"any_codec"`.
  - `"recommended"` (optional): recommended settings that become the defaults when using this service.
    - ~~`"output"`~~: removed and replaced by the `"url"` prefix or `"protocol"` in `"servers"`.
    - `"keyint"` (optional): is now per video codec.
      - **`"any_codec"`** (optional): keyframe interval for any codec.
      - ***`"codec name"`*** (optional, requires to be put in `"supported_codecs"` firstly): (e.g. `"av1"`) keyframe interval for this codec. It overrides `"any_codec"`.
    - `"bframes"` (optional): is now per video codec.
      - **`"any_codec"`** (optional): b-frames for any codec.
      - ***`"codec name"`*** (optional, requires to be put in `"supported_codecs"` firstly): (e.g. `"av1"`) b-frames for this codec. It overrides `"any_codec"`.
    - `"profile"` (optional): is now per video codec.
      - No `"any_codec"` because profile is surely different between codecs.
      - ***`"codec name"`*** (optional, requires to be put in `"supported_codecs"` firstly): (e.g. `"av1"`) profile for this codec if available.  It overrides `"any_codec"`.
    - `"x264opts"`: no change.
    - ~~`"max video bitrate"`~~: replaced.
    - ~~`"max audio bitrate"`~~: replaced.
    - ~~`"max fps"`~~: replaced
    - ~~`"supported resolutions"`~~: replaced.
    - ~~`"bitrate matrix"`~~: replaced.
      - ~~`"res"`~~: replaced.
      - ~~`"fps"`~~: replaced.
      - ~~`"max bitrate"`~~: replaced.
    - **`"fps"`** (optional): recommended framerate for this service. Unused if `"supported_resolutions"` is set with framerates.
    - **`"video_bitrate"`** (optional): recommended video bitrate. Can be set per codec. Unused if `"video_bitrate_matrix"` is set.
      - **`"any_codec"`** (optional): bitrate for any codec.
      - ***`"codec name"`*** (optional, requires to be put in `"supported_codecs"` firstly): (e.g. `"av1"`) bitrate for this codec. It overrides `"any_codec"`.
    - **`"video_bitrate_matrix"`** (optional but requires `"supported_resolutions"` with with framerates): recommended bitrate based on resolutions and framerates from `"supported_resolutions"`.
      - ***`"CX x CY @ FPS"`*** (one per resolution): (e.g. `"1280x720@60":`) recommended bitrate for supported resolution of this service.
        - **`"any_codec"`** (optional): bitrate for any codec.
        - ***`"codec name"`*** (optional, requires to be put in `"supported_codecs"` firstly): (e.g. `"av1":`) for this codec. It overrides `"any_codec"`.
    - **`"audio_bitrate"`** (optional): maximum audio bitrate. Can be set per codec.
      - **`"any_codec"`** (optional): bitrate for any codecs.
      - ***`"codec name"`*** (optional, requires to be put in `"supported_codecs"` firstly): (e.g. `"av1":`) bitrate for this codec. It overrides `"any_codec"`.

##### Advantages of this format

- Recommended settings are really recommended settings.
- Recommendation and maximums are two separated thing.
- Some settings are now per protocol or per codec, this allow multi-protocol service to be registered under only one id. This will also need to a way to know which codec is used when applying settings.
- Name can be changed without consequences.

### Service ID naming scheme
- Only lower case letter
- Hyphen `-` are only used to distinguish two service providing the same stream service but with noteworthy differences like NicoNico (free & premium).

### Things to be reworked
- Verification with video and output settings to see (if not ignored) if the settings for the service are respected.

### About codecs
Some services could support a protocol without supporting all the compatible codecs, so adding a new field like `supported_codecs` for audio and video may be needed.

# Drawbacks

# Additional Information
