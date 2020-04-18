- Start Date: 2020-04-17
- RFC PR: #19
- Mantis Issue: N/A

# Summary

A method for plugins and scripts to notify the user about important or time-critical information. I'll use the term "Alerts" for the rest of the RFC to cover this functionality. This RFC is mostly around the backend design, with elements of the frontend and UX considerations.

The end goal is an alert widget of some description, with a panel/window for in-depth and historical alerts.

# Motivation

Internally, OBS does a lot of things at once, and any one of them can fail for a variety of reasons. Currently, the only way for a plugin to document that something went wrong is to log it. Then, someone with more experience requests and reads the log, to then provide the user with an actual solution. This is a whole lot of unnecessary work for both the user (who is more likely to give up) and our support team.

## Example Alerts

> **Some sources failed to load.** It looks like a plugin may be missing. <a href="#">Read more</a>.

> **Some sources are pointing to missing files.** They may have been moved or deleted. <a href="#">List problematic sources</a>.

> **Output aspect ratios don't match.** This can result in stretched recordings. <a href="#">Fix it</a>.

> **Mismatched Sample Rate found** This can result in audio desync, crackling, etc. <a href="#">Fix it</a>.

> **You just finished a 4 hour stream.** We detected some issues that can be fixed. <a href="#">Check via the analyzer</a>.

> **You were disconnected from your streaming service.** You can read our <a href="#">troubleshooting guide</a> for common solutions.

> **OBS is having trouble maintaining 60 fps.** You can solve this by running OBS as Administrator. <a href="#">Not sure how?</a>

> **An OBS update is available!** You can update manually at any time. Read the full <a href="#">list of changes here</a>.

> **Display Capture requires extra configuration.** You can follow our <a href="#">Laptop Guide here</a>.

> **Issues going live?** You can keep an eye on <a href="#">@TwitchSupport</a> for realtime updates.

# Detailed design

Alerts can have multiple levels based on urgency. Critical, Warning, and Info. In the main window, Alerts are shown one at a time rather than stacking. They can either be dismissed automatically (for example, when an issue goes away or the user fixes the issue manually), or by dismissing it.

## UX Notes

Visually, the Alert will have a coloured background depending on the priority, and within the container an icon to represent the priority, the summary, an 'expand' button to show the full description, and a dismiss button. If expanded, we can provide a direct link to a full list of Alerts. This could either be shown above the preview, or even as part of the status bar.

A full, scrollable, list of active Alerts can be viewed in a secondary window, including Alerts that the user dismissed but are still active (as in, the problem hasn't been solved). From here, the user can explicitly decide to enable/disable alerts from specific plugins. For example, "never show invalid audio device errors". This page could also have methods of filtering/sorting by priority/date, etc.

The summary should be capable of HTML-anchor style actions, allowing the capability to open external webpages or even open specific windows within OBS - say, the Properties window for a specific source, or the Settings window pointed to a specific tab with a certain setting highlighted.

Note: This Alert system would not expose a way to pop up warning/error dialogs. That is not its purpose.

## Internals

The Alerts framework would likely have to be built as part of libobs, providing a pathway from plugins to the UI.

When generating an Alert, the following (bare minimum) parameters should be available:

```js
{
  "timestamp": 1585817608, // An epoch timestamp of when the issue occurred. Generated automatically
  "priority": "CRITICAL", // Enum of CRITICAL, WARNING, INFO // Required? Could default to warning?
  "summary": "", // Required. HTML-anchor capable. A short summary of the issue, visible from the main window
  "description": "", // Optional. A lengthy, more informative HTML string providing more context and inline solutions. Might support newlines and dot points
  "persist": true, // Optional, default to false. If the user dismisses the alert, whether it should persist in the full list. Useful for alerts that are tied to a fixable issue.
}
```

Creating this Alert should return a unique identifier, so that the plugin can keep track and control the status of an Alert. Alongside this, we should also provide an ability to tie Alerts to a specific source (scene? group?), property, and its valid state. That way the plugin relinquishes control of the Alert and it's dismissed as part of the property becoming valid, or the source being deleted/unloaded (when switching Scene collections or closing OBS).

Could be designed similar to hotkeys in terms of API. Examples: `obs_add_source_notification` / `obs_add_frontend_notification` / `obs_add_output_notification`.

Examples

# How We Teach This

On first launch, it'd be nice to show an initial bit of information via this system to introduce the user to OBS, and to explain the purpose of the Alert UI.

For developers, making sure this is well documented is important. Knowing that they can reach out to the user within the UI when things go wrong is important.

# Drawbacks

Complexity. This will require a lot of code, logic, and a background runner keeping track of active/dismissed alerts. The solution to this may be the option to turn it off somehow for bigger productions that only need issues confirmed before going live.

Visual noise. This one is less of a concern, but at the same time we don't want too much going on in the UI. A big toolbar along the top could be distracting, but at the same time going subtle means that the user could potentially not realise that something has gone wrong.

# Additional Information

Alerts shouldn't have a timeout, so that they're not missed by the user.

There are many things to consider in this design. It's highly likely that the backend & UI would be developed & PR'd independently rather than together. This'd ensure a solid backend design, and an easier time reviewing it.

The backend design & scope should be fleshed out significantly more before a real impelementation is started.

# Alternatives

- Exposing QSystemTrayIcon's `showMessage` - providing plugins a way to use native notifications.
  - Downsides include:
    - They will trigger a sound that the user cannot mute
    - Windows automatically disables these when a game is running (via Focus assist)
    - they are distracting and interrupt the user
    - Due to how they're visually exposed to the user, they could be abused by plugins ("annoying")
  - Upsides include:
    - easier to implement
    - will get the user's attention

# Unresolved questions

* Should Alerts have a list of methods to dismiss (scene collection switch, profile switch)?
  * Methods to dismiss would be quite limited in libobs, and would add unneeded complexity to the frontend. Methods to dismiss would instead be tied to sources, outputs, etc.
* The focus of this current design is of the assumption that alerts are only shown in the main window. Often it makes more sense to show warnings/info within Properties themselves. I personally think the scope should be limited to the core alert system, and overhauling Properties to support many more things is a task for another day. Thoughts?
* Rather than making the summary/description link/anchor capable, should we instead provide a "link" field, and auto generate a "Learn more" link at the end of the summary? The big downside of this is then only one link can be provifded, rather than providing extensibility and further information for the user.
* Should this Alert API contact an OBS server to recieve alerts? Use case here would be if a service is having connectivity issues that is out of our control. This could explicitly be optional, maybe only enabled when the user links their account with a service.