- Start Date: 2019-12-25
- RFC PR: #3
- Mantis Issue: N/A

# Summary

Create a new "Browser" transition within the `obs-transitions` plugin that harnasses CEF (via `obs-browser`) as its backend. Basically a "Stinger" but instead of a 'Media Source', it's a 'Browser' source.

# Motivation

The web browser is a powerful tool on its own. As a source in OBS, it's great for building dynamic overlays.

However, JavaScript developers are used to being able to control everything. In this case, a source cannot (and shouldn't) do anything related to transitioning from one scene to another. This is where a Browser transition makes sense, especially as they expose access to `transition_start` and `transition_stop` events.

# Detailed design

## User UX

1. In OBS, under "Scene Transitions", click the +
2. Select "Browser"
3. Populate the Properties
4. Click OK
5. Switch Scenes

## Web Developer Experience

A web developer should have access to all properties/options/events that make sense. See the API section below.

## Properties (basically same as Stinger)

- URL / Local File
- Transition Point Type (ms) & Transition Point
	- Note: it may be worth having a toggle here:
		- OBS announces the transition point to the webpage vs
		- The webpage tells OBS when to switch scenes
- Audio Monitoring
- Audio Fade Style

## API

Ideally, a Transition should have access to `window.obsstudio` same as browser sources & panels.

### Events

- `transition_start`
- `transition_end`
- Potentially some/most [Source Signals](https://obsproject.com/docs/reference-sources.html?highlight=transition#source-signals). The shortlist can be decided later.

### Functions or static values(?)

- `transition_duration` (via `obs_frontend_get_transition_duration()`)
	- This would likely be best as a static/constant JS variable, set automatically when the source loads/`transition_start` is triggered
- `obs_source_info.get_properties`

# How We Teach This

Documentation is key. Ideally I'd like to see the native JS APIs (this one, plus the existing Source and Panel APIs) documented in obsproject.com/docs

# Drawbacks

Users may get confused seeing "Browser" for both Sources and Transitions, so alternate terminology may be required.

# Alternatives

In terms of _current_ alternatives, developers are limited. They can either keep everything in one scene and build a transition into their singular overlay, or tie their overlay into the OBS-websockets API and trigger a transition that way, ensuring the browser source in question is in both scenes for a smooth look.

In terms of future alternatives, developers may make use of a DSK (Downstream keyer), though that doesn't cover the lack of global `transition_start` event which is what `obs-transitions` are designed for.

# Unresolved questions

- Is it possible for `obs-transitions` to correctly expose the JS object `window.obsstudio` the same way browser sources and panels do?
- Should browser transitions have similar manual controls to a regular browser source, like custom FPS and CSS?
- What other Frontend [events](https://obsproject.com/docs/reference-frontend-api.html?highlight=started%20recording#c.obs_frontend_event)/[functions](https://obsproject.com/docs/reference-frontend-api.html?highlight=started%20recording#functions) should be accessible?
