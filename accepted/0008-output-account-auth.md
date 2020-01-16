- Start Date: 2020-01-16
- RC PR: #8
- Issue: N/A

# Summary

Replace the current output-account-auth system with one that allows a modular and extendable system.

# Motivation

At the moment the current system requires that obs-studio/UI is modified to accomodate each new output that has a slightly different setup, such as new docks, non-streamkey authentication, etc. This has several drawbacks:

- Output plugins aren't backwards compatible and can only work with a very specific version of obs-studio.
- External services (YouTube, Twitch, ...) rely on core OBS maintainers to update OBS Studio in a given time.
- No way to extend this outside of making a PR to obs-studio/UI

Instead of that, a modular and extendable design would allow for plugins to simply be installed into obs-studio and remove the workload from OBS maintainers entirely.

# Detailed Design

## Redesign the Output/Encoder tab into a single unified tab:
* Unified design has a list of created outputs at the top, with an add new button to add new outputs.
* Each Output has its own page to configure output settings, audio settings, and video settings.
    * Audio and Video should optionally just be copyable from another output.
* Recording output is always there as it is a core feature.

## Split the Authentication part from the Output/Account API:
* Authentication plugins must have a unique id (no two OAuth providers).
* Authentication must be stateless, any state data (OAuth id, etc) must be stored in an obs_data_t object if it is meant to persist.
* Has a single function, `obs_data_t* /*result =*/ authenticate(obs_data_t* state_storage, obs_data_t* settings)`:
    * Deals with authentication synchronously or asynchronously, but must be blocking.
    * state_storage contains all current or past state to be used for authentication.
    * settings contains any necessary settings, and is provided by the Output/Account plugin.

## Enhance the Output/Account API:
* Add an option to specify a list of preferred authentication providers.
    * List may be nullptr/empty if no authentication is needed.
    * This is the case for direct RTMP/url streaming which just needs a streamkey.
* Allow providing settings via the already established properties API.
    * Optionally allow frontend plugins to override this behavior.
    * Crashing is invalid behavior if said frontend-plugin is missing, but not loading is a valid action.
* Add a capability to specify multi-output capability (OBS_OUTPUT_CAP_MULTI).
* Add a category and classification field for the UI.
    * Allows for the UI to hide 18+ services to comply with EU and US child protection laws.
    * Category field allows the UI to further categorize outputs, but is optional.
* Add an icon/image/logo field.
    * Must point to a local file.
    * Is used by UI to show a proper logo instead of just the output name text.
* Add a new get_description function to allow outputs to specify to a user what they are.
* Add a field/function to get a list of supported audio and video codecs.
    * Special entries for buffer/texture passthrough (no codec support) for things like WebRTC.

## Enhance the FrontEnd API:
* Allow overriding for specific settings UIs in output, audio and video plugins.

## Add a new Guide window for creating a new output:
* Uses the category and classification fields from outputs to determine if an output should be shown.
* When an output is clicked/selected, shows the description provided by the output.
* Allows for full-text searching to find a specific output.

## Notes:
* FFmpeg Output is now split from Recording but is still considered a recording output. Both standard recording and FFmpeg can exist in parallel.
* Recording and Replay Buffer are still unique, you can't have more than one.
* Profiles are still perfectly valid if the end user has to choose which outputs to stream to.

# How We Teach This

- Q: How do I add a new stream output?
A: Click 'Add New' in the Output page in the Options, then select which type of output you want.

- Q: Can I still re-use an encoder from another output?
A: Of course. Simply select '(Use OUTPUTNAME Encoder)' and it will use that encoded stream for this output as well.

# Drawbacks

"Simple" Output mode will be vastly different and can now be specified per output.

# Alternatives

None.

# Unresolved Questions

- Which platforms/outputs allow for multiple outputs?
    - Should we enforce their rules?
    - What about platforms where it isn't clear?
- Should scaling settings be provided per-output?
    - Massive CPU/GPU cost if done so
    - Can this even be avoided?
- Should we allow a UI override to specify which scene is sent to which output?
    - Fixes another user request.
    - Complex behavior, but is very versatile.
    - Possibly better to allow multiple active "masters"?
- Should the UI allow choosing which outputs are tied to which button?
    - Complex behavior, but is very versatile.
    - Doesn't fix any known user requests, but might fix a future user request.
    - Alternative: Add a field to configure which button an output is tied to, including the creation of new buttons.
