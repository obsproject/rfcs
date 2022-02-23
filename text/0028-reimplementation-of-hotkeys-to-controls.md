# Summary
* Remove completely the current "hotkeys" implementation
* Creation of a new "Controls" menu in settings
* Creation of two new plugin types "Triggers" and "Actions"
* Creation of a default Action Plugin "OBS"
* Creation of a default Trigger Plugin "Hotkeys"
* Creation of a default Trigger plugin "OBS"
* (optional) Creation of a default Action plugin "Macros"


# Motivation
Currently we have the ability to control obs through plugins, There is no unified structure that allows for those controls to be accessed in one location, or to communicate with each other.

This RFC is in regards to building a framework to allow for multiple Triggers (eg. Hotkeys, Midi, OSC, Xinput, etc.) to fire off arbitrary actions (OBS, MIDI, OSC, Force Feedback)

# Detailed Changes

## Overview of changes
Several changes need to be made to support this new feature, including UI, plugin module,and possibly configuration. The high-level idea of how we plan to implement this feature is as follows:

1. Replace hotkeys in the settings window with a controls page
2. Create a plugin type that allows for tabs to be added to the controls page
2. Reimplement the hotkeys functions as an input type control plugin
4. Create a generic control message that can is sent to the mapped action when a Trigger is called
3. Create a Control manager that handles 
   * Saving and loading Trigger/Action mappings
   * Message passing between Triggers and Actions

## Plugin type specifics
### Both 
The control plugin type needs to have the following features, 

* Define whether it is a Trigger Plugin, or an Action Plugin
* A QtWidget to add a configuration page as a dropdown to the Control Menu if needed for the protocol
* Add A custom UI to the "mapping page" when the plugin is selected in the QComboBox


### Trigger Plugins

A Trigger plugin is used to receive an input or event and send a message to the Control Manager

### Action Plugins
An Action Plugin, is used to carry out an action, whether that is to switch a scene in obs, or Provide MIDI feedback 


## Potential plugins that could use this system

### > Trigger
* Hotkeys
* MIDI
* OSC
* Joystick
* MQTT 
* OBS Events
### > Action
* Midi
* OSC
* MQTT 
* Macros

# UI 
A rough idea of a possible ui

## Control Menu

![new Hotkeys tab](https://github.com/cpyarger/rfcs/blob/master/text/basic%20settings%20idea.png?raw=true)
## MIDI
### Midi Device Tab
![new midi device tab](https://github.com/cpyarger/rfcs/blob/master/text/midi%20tabs.png?raw=true)

### Midi Mapping Page
![new midi mapping page](https://github.com/cpyarger/rfcs/blob/master/text/midi%20mapping.png?raw=true)




# Potential Drawbacks

* Improperly implemented controls can cause control feedback loops
* Possible slowdowns if a large number of calls happen very fast (a few hundred or thousand / second)
