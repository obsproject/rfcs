# Summary

Blackmagic devices that use the DeckLink SDK provide much richer information about the status and configuration of the devices than we are currently leveraging, we should integrate with the required APIs for this to later build better UIs. OBS also doesn't keep track in a single place what OBS sources and outputs are using devices and we can't tell the user when there are conflicts, we should build a way to show this information.

# Motivation

Currently we don't play very nice if there is another app using a DeckLink device and can either stomp on them or just fail to start an input or output. We also don't have a way to give the user feedback if their settings in OBS are incompatible with the configuration and or current state of the device especially for devices like the DeckLink Quad 2 and the DeckLink Duo 2 that support sub-devices. This should also help us build out a more robust UI for configuring outputs that show things like multiview and specific scenes.

# Implementation

To get information on sub devices we will need to pull data from `BMDProfileID` to know how the connectors are setup and what is an input, output, or key fill.

For device states we will want to subscribe to `bmdStatusChanged` and update information based on the `bmdDeckLinkStatusBusy` status.
There are probably other configuration states we will want to subscribe to here too.

To make sure that devices don't have conflicts we will want a way to keep track of all the configured decklink sources across scenes and track what devices they are using. With this information we can tell users when inputs in the same scene will conflict, and also when an output and input could possibly conflict. (Will all the sources be loaded already or do they get unloaded when we switch scenes? If they all run, we can just pass this info up to a top level decklink manager class)

This will not be building out the new UIs just providing the primitives to get us to a place where we can build them.

# Drawbacks

* There will be additional complexity 
* Gathering data from a bunch of sources / outputs could have some unforeseen challenges


# Additional Information

https://github.com/kdienes/decklink-sdk/blob/master/Blackmagic%20Decklink%20SDK.pdf