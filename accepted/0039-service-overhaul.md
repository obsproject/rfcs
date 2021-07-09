# Summary

- Make OBS able to accept third-party service plugins
- Attribute a id for each services rather than a common one
- A service can be provided with multiple protocols
- Separate Twitch and Restream integation and make them plugins

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

# Detailed design
***This RFC is dependent on RFC 38.***

## UI
The Stream settings page will only contain a combo box listing registered services with a remade "Show All" option. So property view can be shown under it.
The "short list" with "Show All" option, will be harcoded with a list of service id in the page code.
Any removed element will be reimplemented in a new way.

(Same for auto-wizard)

## Service plugins
 ### `rtmp-services`
 This plugin will be put in a state of deprecation with a legacy behavior to not break immediately every setup. And be replaced with the three following plugins.

 Since OBS is heavily dependent of this plugin, many changes will be needed to remove this dependency.

 The legacy behavior could reinterpret output set in recommended as protocol to make it compatible with the new aproach.

 ### `obs-services`
 This plugin will use a brand new services.json (with a new format), to register each streaming service with their own id. No more things like `rtmp-common` id.
 
 Like this if the service needs a specific behavior, you can 'transfer' it to `obs-services2` and keep the same id. The change will be seamless for the user.

 Those services will be able to provide multiple protocols so no more "Service - HLS" and "Service - FTL".

 If a certain protocol is not availlable, the plugin will not register the service if the service doesn't use another protocol.
 If it does, the missing protocol will not be shown. Thanks to RFC 38.

 Maximum, supported resolutions, recommended settings will be ignorable with their respective checkboxes.

 Those services should have no specific behavior like ingest management. But can list with which codec they are compatible if needed.

 In the end, adding streaming service will only be a JSON addition.
 Only improvements will be accepted in the code of this plugin.

 ### `obs-services2`
 This plugin is meant to have implemented and register services which need a specific behavior like ingest with a unique id for each one.

 No JSON file will be used to create those services, except maybe for servers list.

 ### `custom-service`
 This plugin is meant to provide a replacement for `rtmp_custom` type.

 If OBS need to be heavily dependent to one plugin, it shall be this one.

 The protocol will be detected (or maybe forced set by the user).
 Maybe properties will show depending of the protocol.

 ### `obs-twitch`
 This plugin is meant to restore Twitch integration with a different id from the original service.

 ### `obs-restream`
 This plugin is meant to restore Restream integration with a different id from the original service.

## API changes
- Add missing get_properties2 and get_default2 functions for service
- Add protocol property to service
- Add a property to be able to show information like string with the supported resoltions
- Add a property to be able to show a information in kbps like maximum bitrate for a service
- Add a property to be able to show information in FPS like maximum FPS for a service
- Add a property to add a button with an URL to provide "More Info" and "Get Stream Key" button

Since the service API is in a way unusable by third-party, any breakage will only impact `rtmp-services`.

## New service JSON format **Needs RFC 38 ajustments**
Here is a example:
```json
{
    "format_version": 4,
    "services": [
        {
            "id": "example",
            "name": "Example of stream services",
            "more_info_link": "https://example.com/more_info",
            "stream_key_link": "https://example.com/stream_key",
            "available_protocols": ["RTMP","FTL"],
            "servers": [
                {
                    "name": "Example RTMP 1",
                    "url": "rtmp://example.com/server1"
                },
                {
                    "protocol": "RTMP",
                    "name": "Example RTMP 2",
                    "url": "rtmp://example.com/server2"
                },
                {
                    "protocol": "FTL",
                    "name": "Example FTL",
                    "url": "ftl://example.com/server_ftl"
                }
            ],
            "maximum": [
                {
                    "protocol": "RTMP",
                    "video_bitrate": 3000,
                    "audio_bitrate": 320
                },
                {
                    "protocol": "FTL",
                    "fps": 30,
                }
            ],
            "supported_resolutions": [
                "1920x1080",
                "1280x720",
                "852x480",
                "480x360"
            ],
            "recommended": [
                {
                    "protocol": "RTMP"
                    // Work In Progress
                }
            ]
        }
    ]
}
```

In this new format:
- Each service will also have his own id and be registered with it.
- Link for more info and stream key can be added.
- Multiple protocols can be set for one service, if not it will default to RTMP only.
- Servers have now the attribute "protocol", if not provided it will default to RTMP.
- Maximums are now separated from recommended settings and can be set for each protocol but no settings for any protocol. **And the protocol shall be set, there will be no fallback to RTMP**
- Supported resolutions are separated to, it's usualy because of the player used by the service, so it's set for any protocol
- Recommended settings and can be set for each protocol but no settings for any protocol. **And the protocol shall be set, there will be no fallback to RTMP.** *Consider it as futureproofing for when new codec for other protocol like AV1 will come. RTMP spec tell that it only support H264.*

### Service ID naming scheme
- Only lower case letter
- Hyphen `-` are only used to distinguish two service providing the same stream service but with noteworthy differences like NicoNico (free & premium) or Twitch and Twitch OAuth

OR

- Use the Reverse Domain Name notation
- And add a sub-domain element to distinguish two service providing the same stream service but with noteworthy differences like NicoNico (free & premium) or Twitch and Twitch OAuth

## VOD Track
This option will only be shown if the combo service-protocol is compatible.

## Things to be reworked
- Verification with video and output settings to see (if not ignored) if the settings for the service are respected.

# Drawbacks

This will break old OAuth authentification.

# Additional Information

You can find a draft about `obs-services` [here](https://github.com/tytan652/obs-studio/tree/service_api/plugins/obs-services).
