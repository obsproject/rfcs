- Start Date: 2020-01-23
- RFC PR: #10
- Mantis Issue: N/A

# Summary

Make OBS bind to the `obs://` URI handler in order to receive configuration from web pages and other applications. As an MVP this would allow other services to communicate RTMP URL and RTMP Streamkey to OBS, the design allows for other parameters to be added to this in the future while remaining backwards compatible. It could for example be extended to allow one-click installation of plugins in the future.

# Motivation

The current setup process from streaming service to OBS is a bit rough for new users, as they must paste a stream key and configure the RTMP URL correctly. The list of services keeps growing and if new services want to onboard onto this list the list will eventually become too long to be meaningful to users.

On top of this streaming services today treat the streamkey as something persistent due to the friction of reconfiguring the streaming settings. Enabling a easier user flow would allow streamkeys and configurations to be more ephemeral, generating a new one per streaming session for example.

# Detailed Design

## URI Design

The URI Scheme would be `obs://ACTION/VERB?=parameters&=to&=add`

Motivation behind this scheme is that you can build the simplest implementation of configuring streaming service like this:
`obs://configure/streaming?rtmpurl=rtmplive.twitch.tv&streamkey=heregoesmynicestreamkey`

It also allows for plugin installation like this in the future:
`obs://install/plugin?source=somewhere.com/plugin.zip&signature=asdf123asdf123`

Same with allowing for automatic configuration of the browser source for overlay providers such as StreamElements and StreamLabs:
`obs://scene/add?source=browser&url=something&width=1920`

## User UX

1. User clicks URI
2. OBS receives URI callback
3. OBS shows confirmation for users “Do you want to configure streaming service to X?”
4. User confirms and OBS accepts settings

## Design Stories

**As a user**

I want to be able to quickly configure OBS without having to

* Manually copying the streamkey
* Understanding which part is the RTMP URL or the streamkey

When I click a link to configure OBS

* I want to a confirmation dialog to confirm my intent
* I want the settings to create a new profile in OBS to not delete my current settings

**As a streaming service**

I want to be able to help my users to configure OBS

* Without adding code to OBS and add my streaming service to the large list
* Without adding a detailed guide on how to configure OBS

I want to able to generate these links programatically

# Drawbacks

By making the URI handler be obs:// other software cannot reliably bind to this compared to using a URI like streaming:// . However using a custom URI rather than a generic allows for more integration actions with OBS long term.

# Alternatives

Currently the way to help users configure OBS is by providing step by step visual guides on how to enter the stream settings correctly into OBS. Issue with this is that users commonly paste the wrong part of the settings on OBS.
Another method is distributing a JSON configuration and asking users to copy this into the correct folder for OBS to read it, this ends up being harder than the other options of configuring OBS.
