- Start Date: 2019-12-30
- RFC PR: #4
- Mantis Issue: N/A

# Summary
Create a plugin manager that is capable of configuring the loading of plugins as
well as providing an easier means of installing plugins. Perhaps - later down
the line - creating a online database that individuals can search and seamlessly
install plugins.

# Motivation
Varying from plugin to plugin, there are a variety of different intallation
methods for each plugin. On top of that, once installed, there is no easy way to
disable, or even configure, each individual plugin. In addition, if one plugin
doesn't behave correctly because of another, it can become difficult to diagnose
which plugins are causing the error. By implementing a plugin manager, it allows
users to seamlessly install plugins and have finer control over the loading of
their plugins. Furthermore, having a plugin manager opens up many more
possibilties for both users and developers.

# Design
## Functionality
### Loading
Currently, when plugins cause issues, the only way to disable them is by
outright uninstalling them. However, this could be mitigated at startup by
having an internal record of which plugins are installed. By keeping a list -
perhaps in the configuration file - of the disabled plugins, they could be
ignored during initilizaiton.

Furthermore, if a plugin causes a major failure while loading or doing use, it
will be automatically disabled on the next startup with a popup warning the
user. The popup will allow the user to re-enable the plugin (perhaps they
knowingly used a experimental feature, etc.) or keep the plugin disabled. It
would also be helpful to show an error code to the user to help them and the
developer debug.

### Installation
To further ease the user with their plugin (once downloaded), installation could be handled in 3
different ways.
- A file explorer in the manager to install a compressed package that contains:
  a manifest, locales, the actual plugin, and other needed components.
- A file explorer that installs plugins by themselves - mostly for compatibilty
  with previous plugins
- A compressed package that is associated with OBS so when double clicked, it
  can be automatically installed in the correct locations.

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
Plugins. The plugin manager itself would have a searchbar at the top that allows
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
plugin, it will attempt to initialize the plugin.

If the plugin has its own settings menu, the information could be stored in its
manifest which would be unpacked and installed.

### Future Considerations
It would also help plugins be discovered if there was a built in browser for
searching through plugins on a server. A user could search by name or other
metadata (again outlined in the manifest) which would be in another list. Then,
they could click on an install button next to the options to automatically
download and install the plugin.

# How We Teach This
It would be provided in the documentation. However, the bare functionality would
hopefully be designed well enough that it can become plainly clear to the user
how it works.
