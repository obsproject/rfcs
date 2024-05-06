- Start Date: 2019-12-30
- RFC PR: #4
- Mantis Issue: N/A

# Summary
Create a plugin manager that is capable of configuring the loading of plugins as
well as providing an easier means of installing plugins. Furthermore, the plugin
manager should be responsible for updating said plugins. This should be achieved
by hosting a platform for developers to submit their plugins that can be
verfied, signed, and easily maintained. More about managing and the necessary
api in policy.

# Motivation
Varying from plugin to plugin, there are a variety of different intallation
methods for each plugin. On top of that, once installed, there is no easy way to
disable, or even configure, each individual plugin. In addition, if one plugin
doesn't behave correctly because of another, it can become difficult to diagnose
which plugins are causing the error. By implementing a plugin manager, it allows
users to seamlessly install plugins and have finer control over the loading of
their plugins. Furthermore, having a plugin manager opens up many more
possibilties for both users and developers.

More importantly, however, is the need for better discoverability and
compatibilty. As OBS undergoes changes and updates, it could easily break
different plugins. The plugin manager would be responsible for allowing users to
better find the plugins they need as well as ensuring that they work with their
version of OBS. To further reinforce compatability, their should also be
different levels, such as stable, edge, and beta, again, more discussed in policy.

# Design
## Functionality
### Loading
Currently, when plugins cause issues, the only way to disable them is by
outright uninstalling them. However, this could be mitigated at startup by
having an internal record of which plugins are not to be run. By keeping a list -
perhaps in the configuration file - of the disabled plugins, they could be
ignored during initilizaiton.

Furthermore, if a plugin causes a major failure while loading or doing use, it
will be automatically disabled on the next startup with a popup warning the
user. The popup will allow the user to re-enable the plugin (perhaps they
knowingly used a experimental feature, etc.) or keep the plugin disabled. It
would also be helpful to show an error code to the user to help them and the
developer debug.

There should also be a "safe mode" where all plugins not officially apart
of the OBS plugin repository, "3rd party", would be disabled.

### Installation
To further ease the user with their plugin (once downloaded), installation could be handled in 3
different ways.
- A file explorer in the manager to install a compressed package that contains:
  a manifest, locales, the actual plugin, and other needed components.
- A file explorer that installs plugins by themselves - mostly for compatibilty
  with previous plugins
- A compressed package that is associated with OBS so when double clicked, it
  can be automatically installed in the correct locations.
- A search that allows the client to connect to an OBS officiated server (by
  default and can be customized with 3rd party servers) and automatially install
  the plugin.
  
  If the user installs a compressed package, the package would be decompressed and
  the contents would be copied to a directory in the plugins folder.

### Manifest
As outlined in installed, a compressed package will contain multiple components
including a manifest. The manifest will be responsible for outlining all
important information about the plugin. An example of a barebones manifest is
shown below:

```json
{
    "name": "Plugin Name",
    "author": "Author Name",
    "description": "Description that would be displayed to user",
    "locale": {
        "en": "file path"
    },
    "plugin": {
        "64": "file path",
        "32": "file path"
    },
    "categories": ["PRODUCTIVITY", "CLASSIC", "THEMES"], //etc
    "version": "^24.0.0"
}
```

The purpose of the metadata in the manifest is for 3 main reasons: 1) Allowing
for easier parsing of the compressed package, 2) allowing more flexibility for 
plugin creators, and 3) reducing mental overhead for users organizing their
plugins.

Some notable entries on the manifest are categores and version. Version
numbering allows the plugin creator to specifiy a specific version or range that
the plugin can be loaded in. The categories affects the UX.

Note: The options in the manifest are predefined. A plugin developer can not
have random options that can be searched as outlined in UX.

### UX
To access the plugin manager, it would be under Settings -> Plugins or File ->
Plugins or a quick link in the toolbar. The plugin manager itself would have a searchbar at the top that allows
searching through the metadata as provided in the manifest file and displays the
results in the list below. Moreover, there would be a simple list view that
outlines all the installed plugins in alphabetical order. The list would show
the name, description, and categories of each plugin. All the way to the right,
would be a button that outlines whether the plugin is disabled or currently
active. Another feature would to allow the plugin to have its own settings menu
which would display a button next to the disable button that opens its settings menu.

When a user disables a plugin, it will add the plugin to a list to prevent
loading during initilization. It will also attempt to unlink the plugin, but a
restart might be necessary. Likewise, when a user re-enables or installs a
plugin, it will attempt to initialize the plugin. As noted, the current climate
does not permit cleanly unloading, so a restart would initially be required.

If the plugin has its own settings menu, the information could be stored in its
manifest which would be unpacked and installed.

### Future Considerations
It would also help plugins be discovered if there was a built in browser for
searching through plugins on a server. A user could search by name or other
metadata (again outlined in the manifest) which would be in another list. Then,
they could click on an install button next to the options to automatically
download and install the plugin.

# Policy
In order to ensure a quiality standard, it should also be discussed on how to
best deliver the plugins through a server and plugin browser. First, there
should be an open api standard developed that would allow clients to easily
search through plugins. Such endpoints could be /plugins/search/{query}. There
should also be a standard authentication. This would allow developers
to create their own 3rd party plugins servers that they can regulate access too.

Once the API is created, there should be a main server and a community server,
much like the arch linux aur repository. The main server would be OBS curated
plugins that can be approved for stability and security. The community server
would be where developers can go to upload their custom plugins, however, it
won't be as tightly regulated so there is more risk for the user. 

An important part of officially regulated plugins (and even plugins on the
community server and 3rd party servers), is that the plugins are digitally
signed. This is to ensure quality, and security.

Another important part of the API would be the stability of the plugin. This
would tell users what to expect when they install the plugin, and give ample
warning on what plugins _could_ be causing issues. It should be divided into
several categories: stable, edge, beta, alpha, etc.

# How We Teach This
It would be provided in the documentation. However, the bare functionality would
hopefully be designed well enough that it can become plainly clear to the user
how it works.
