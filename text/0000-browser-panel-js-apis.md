# Summary

Provide the [JS Bindings API described in the obs-browser
readme](https://github.com/obsproject/obs-browser#js-bindings) to browser-panels
(docked UI panels). 

# Motivation

OBS users can embed web UI in OBS that allows them to see information related to
their broadcast while they are streaming (e.g. Twitch Chat, Stream Statistics).
However embedded UI does not have access to respond to OBS events or control
OBS. 

Developers have requested JS APIs for UI panels previously in [GitHub
comments](https://github.com/obsproject/obs-browser/pull/312#issuecomment-975668271)
and on ideas.obsproject.com: [Allow use of obs javascript apis in Custom
Browser
Docks](https://ideas.obsproject.com/posts/1623/allow-use-of-obs-javascript-apis-in-custom-browser-docks).

I'd like to create an embedded web UI that can add and remove browser sources
from a scene to streamline the experience of using Twitch's Guest Star features.
Unifying the JavaScript API bridge between browser sources and UI panels could
be the first step to enable this and other future use cases.

# Drawbacks

* Opening new APIs to UI panel creates opportunities for security issues where
  they were previously isolated to browser-sources.
* This change requires permissions settings for UI panels where there were none 
  before.

# Additional Information

## Alternatives

**Configure obs-websocket when setting up a service and provide a means to
exchange the port and password.**

The OBS websocket plugin provides the APIs needed to control OBS from anything
that can connect to the local websocket. An alternative workflow for creating
scene sources from a UI panel would require users to copy and paste the port,
password, and IP address into UI to allow it to control OBS. This requires more
setup from users and creates an opportunity to break the UI connection (changing
any of the values or disabling the server).

If the websocket plugin could be configured during service setup (sharing the
password with the service) and there were some way to detect changes in the
websocket that would break the connection then this could be a viable
alternative.

**The [custom URL scheme RFC](https://github.com/obsproject/rfcs/pull/10) could
allow UI panels to control sources.**

I'm not sure if the current web browser would let OSB users acknowledge that the
app is open. However this wouldn't provide the two way communication (i.e.
events are missing) to keep OBS UI in sync with a UI panel.

## Proposal Specifics

In this RFC I'm proposing that we provide JavaScript API parity between
browser sources
([obs-browser-source.cpp](https://github.com/obsproject/obs-browser/blob/master/obs-browser-source.cpp))
and browser panels
([browser-panel.cpp](https://github.com/obsproject/obs-browser/blob/master/panel/browser-panel.cpp)).
This requires an introduction of the "Page permissions" concept to UI panels.
An initial implementation could leave out the UI for managing these permissions,
only opening permissions when panels are added via services.
