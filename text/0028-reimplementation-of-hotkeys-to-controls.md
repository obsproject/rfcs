# Summary
* create new plugin type "control"
* Reimplement hotkeys as a control plugin


# Motivation
While currently we have the ability to control obs through plugins, There is no unified structure that allows for those controls to be accessed in one location, or to communicate with eachother.

This RFC is in regards to building a framework to allow for multiple control modules (eg. Hotkeys, Midi, OSC, Sound Board, Joystick, etc.) 

# Detailed Changes

## Overview of changes
Several changes need to be made to support this new feature, including UI, plugin module,and possibly configuration. The high-level idea of how we plan to implement this feature is as follows:

1. Replace hotkeys in the settings window with a controls page
2. Create a plugin type that allows for tabs to be added to the controls page
2. Reimplement the hotkeys functions as an input type control plugin
4. Create a generic control message type that can be passed between each plugin, probably similar to the frontend callback type
3. Create a Control manager that handles saving and loading IO mappings 

## Plugin type specifics
The control plugin type needs to be able to 
* Handle both input and output control plugins
* Be able to enable and disable plugins via an enabled checkbox  or enable/Disable button in the new ui page


### Input Type control plugins

The input type plugin is used to implement receiving inputs into obs and mapping them to an "output type" plugin
it needs to be able to 
* Create a new item to the list widget on the new "controls page"
* Add it's custom UI to a mutable layout below the list widget the "mapping page"

* Actual control of obs is already possible using existing methodologies. 
### Output Type control plugins
The output type plugin is used to send events from obs, or an "input type" plugin.
Output type plugins get added to a QComboBox within the "input type/mapping page"



## Example plugin types
### Input

* Hotkeys
* MIDI*
* OSC
* Joystick
* MQTT
* (possibly) WebSockets*
* OBS -- onObsFrontendChange -- Gives us a graphical way to map obs events to outputs
### Output
* Midi* 
* OSC
* MQTT
* (possibly) WebSockets*

*Websockets and MIDI already exist with the obs-websockets and obs-midi plugins

# UI 
A rough idea of a possible ui
## Hotkeys
![new Hotkeys tab](https://github.com/cpyarger/rfcs/blob/master/text/basic%20settings%20idea.png?raw=true)

## MIDI
### Midi Device Tab
![new midi device tab](https://github.com/cpyarger/rfcs/blob/master/text/midi%20tabs.png?raw=true)
### Midi Mapping Page
![new midi mapping page](https://github.com/cpyarger/rfcs/blob/master/text/midi%20mapping.png?raw=true)




# Drawbacks

* Improperly implemented controls can cause control feedback loops
* Possible slowdowns if a large number of calls happen very fast (a few hundred or thousand / second)
** I have not yet noticed this with the obs-midi plugin even when doing testing for it (moving multiple volume faders as fast as possible)

# Additional Information
I would like to see the [UI: Redesign hotkeys interface #2133](https://github.com/obsproject/obs-studio/pull/2133) as the new hotkeys ui within the control module
I have been working on the [OBS-Midi](https://github.com/Alzy/obs-midi) plugin with Alzy 
