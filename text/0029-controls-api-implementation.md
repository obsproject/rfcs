# Summary

Either Extend the obs-frontend-api, or create an ops-control-api

# Motivation

For plugins to control OBS, many of the same functions are currently required to be implemented individually by each plugin. My suggestion is to take many of the wrapper functions in use by OBS Websockets for interfacing with OBS and add them to OBS itself so they don't need to be reimplemented by the plugins.

This RFC may be used in conjunction with RFC28, but is completely UI independent.

# Drawbacks


It may add a significant amount of extra code that does not currently exist in OBS
# Additional Information
Both OBS Websockets and OBS Midi use much of the same code for interfacing with OBS,  including pulling data, executing actions and dealing with feedback from OBS
