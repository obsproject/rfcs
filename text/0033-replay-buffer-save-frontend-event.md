# Summary

Add a frontent event to the API that would enable reacting to replay buffer being saved from inside a plugin

# Motivation

Currently the API does not give a possibility to react to the event of the replay buffer being saved from a plugin.
Note that the following 4 events are already available in the C frontend API:

- OBS_FRONTEND_EVENT_REPLAY_BUFFER_STARTING
- OBS_FRONTEND_EVENT_REPLAY_BUFFER_STARTED
- OBS_FRONTEND_EVENT_REPLAY_BUFFER_STOPPING
- OBS_FRONTEND_EVENT_REPLAY_BUFFER_STOPPED

This one missing event will extend plugin API capabilities allowing for more interaction.

Implementation:

Add `OBS_FRONTEND_EVENT_REPLAY_BUFFER_SAVED` to `obs_frontend_event` enum in `<obs-frontend-api.h>` and emit that event in the GUI interaction part

# Drawbacks

There should be no drawbacks other than the ABI change; i.e. the fact that unless the new event is appended at the bottom of the enum, all plugins that make use of it need to be recompiled. 

# Additional Information

None.
