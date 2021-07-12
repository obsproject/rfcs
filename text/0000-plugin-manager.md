# Summary
Proposes a plugin manager for libOBS (OBS Studio, OBS.live, Streamlabs OBS, ...) that eases the installation and removal of plugins into libOBS.

# Motivation
At the current point in time with version 26.1.x, users still have to manually install plugins and search for these. This is usually prone to user errors, and often results in false bug reports and additional support work.

# Design
The Plugin Manager is a plugin to libOBS itself, requiring no changes to the libOBS API itself which may break ABI compatibility between versions. It uses the [procedure handler API](https://obsproject.com/docs/reference-libobs-callback.html#procedure-handlers) to provide other plugins or UI integrations a way to directly interface with the manager.

## Plugin Manager API
### Structure
```
struct obs_plugin_manager {
	/** The API and Feature level that the plugin manager supports.
	 *
	 * Consists of two 32-bit unsigned integers stored with the current CPUs 
	 * endianness, storing the API level in the bits [0...31] and the feature 
	 * level in the bits [32...63].
	 *
	 * The API level increases only when incompatible API changes are made, such
	 * as deleting or moving members. It should be considered unsafe to continue
	 * using the API if a higher or lower API level is found.
	 *
	 * The feature level increases for additions to the current API level and
	 * resets back to zero if a new API level is introduced. It is safe to use a
	 * higher feature level than supported, but not a lower one.
	 */
	uint64_t api_feature_level;

	// API Level 0, Feature Level 0

// -- PROVIDER API -- 
	/** Register a new plugin provider with the manager.
	 * 
	 * @param provider A pointer to a provider for plugins.
	 * @param provider_size The size of the obs_plugin_provider_t structure.
	 * @return true if successful, false if an error was encountered.
	 */
	bool (*register_provider)(obs_plugin_provider_t provider, size_t provider_size);

	/** Count the number of providers currently registered.
	 * 
	 * @return The number of providers registered.
	 */
	size_t (*count_providers)();

	/** Get the provider at the specified index.
	 * 
	 * @param idx 
	 * @return A valid provider if the index is in range, otherwise 0.
	 */
	obs_plugin_provider_t (*get_provider)(size_t idx);

	/** Get the provider described by the given id.
	 * 
	 * @param id A null-terminated string containing the identifier to find.
	 * @return A valid provider if the id was found, otherwise 0.
	 */
	obs_plugin_provider_t (*find_provider)(const char* id);

// -- DEPENCY API --
	/** Register a non-plugin dependency.
	 * 
	 * This should be used to register things like libobs, qt, cef and other 
	 * parts that are not a plugin into the dependency tree.
	 *
	 * @param id The dependency id.
	 * @param version A C-string containing a version in the format "A.B.C.D", 
	 *        with each component matching the regular expression /[0-9]+/.
	 * @param location A UTF-8 C-string that contains the location for the given
	 *        dependency. May be set to 0 to specify a location-less dependency.
	 * @return true if the registration succeeded, false if there was an error.        
	 */
	bool (*register_dependency)(const char* id, const char* version, const char* location);

	/** Check if a dependency can be fulfilled.
	 *
	 * Checks against an internal database (which also contains plugins) if a
	 * dependency can be fulfilled with the currently installed dependencies and
	 * plugins. 
	 *
	 * @param id The dependency id to try and find.
	 * @param min_version The minimum version of the dependency to fulfill,
	 *        failing if no version greater than it was found. This should be a
	 *        C-string with the format "A.B.C.D" where each components matches
	 *        the regular expression /[0-9]+/.
	 * @param max_version Optional, specifies the upper limit for versions to 
	 *        look for. Set to 0 if not needed.
	 * @param location Optional, a pointer to a 'const char*' that will hold the
	 *        location of the fulfilled dependency. Zeroed if it could not
	 *        be fulfilled, or if the dependency has no location.
	 * @return true if the dependency could be fulfilled, false if not.
	 */
	bool (*check_dependency)(const char* id, const char* min_version, const char* max_version, char** location);

// -- PLUGIN API -- 
	/** Register an installed plugin with the manager.
	 * 
	 * This does not perform any loading of the plugin until later, it only 
	 * informs the plugin manager that a plugin is at a certain location. It
	 * additionally registers the plugin into the dependency database for later
	 * fulfillment of dependencies.
	 * 
	 * @param plugin An obs_plugin_information_t object that contains all necessary information about the plugin.
	 * @return true if registering was successful, false if an error occured.
	 */
	bool (*register_plugin)(obs_plugin_information_t plugin);

	/** Count the number of available plugins.
	 * 
	 * Only counts unique identifiers, not versions, locations or providers.
	 * 
	 * @return The number of registered plugins.
	 */
	size_t count_plugins();

	/** Retrieve the plugin id at the given index.
	 * 
	 * @param idx The index to retrieve the identifier for.
	 * @return The identifier of the plugin at the given index, or zero if the index is invalid.
	 */
	obs_plugin_information_t get_plugin(size_t idx);

// TODO
// - Register installed plugins
};

typedef obs_plugin_manager* obs_plugin_manager_t;
```

### Accessing the Manager
The singleton instance of the obs_plugin_manager can be retrieved with the following bit of code:

```
// You need a place to store the actual structure pointer.
obs_plugin_manager_t manager;

// Set up the calldata.
calldata_t* cd;
calldata_init(cd);
calldata_set_ptr(cd, "manager", &manager);

// Call the procedure handler.
proc_handler_call(obs_get_proc_handler(), "obs_get_plugin_manager", cd);

// Free the calldata as we no longer need it.
calldata_free(cd);
```

## Plugin Provider API
Information about available plugins is not shipped with the manager and instead provided by separate services. This mimics the idea behind package registries like NPM, without the single-service restriction that these usually have.

### Structure
```
struct obs_plugin_provider {
	/** The API and Feature level that the plugin provider supports.
	 *
	 * Consists of two 32-bit unsigned integers stored with the current CPUs 
	 * endianness, storing the API level in the bits [0...31] and the feature 
	 * level in the bits [32...63].
	 *
	 * The API level increases only when incompatible API changes are made, such
	 * as deleting or moving members. It should be considered unsafe to continue
	 * using the API if a higher or lower API level is found.
	 *
	 * The feature level increases for additions to the current API level and
	 * resets back to zero if a new API level is introduced. It is safe to use a
	 * higher API level than supported, but not a lower one.
	 */
	uint64_t api_feature_level;

	/** Unique id of the provider.
	 * 
	 * Identifies the provider to prevent multiple versions of the same provider 
	 * from existing. The name should ideally be in a reverse-DNS
	 * structure (a.b[.c[.d[.e[...]]]]) and must match the following regular
	 * expression: `[a-zA-Z0-9\-\.]+`.
	 *
	 * This member must be valid for the entirety of the provider lifetime.
	 * This member must not be zero.
	 */
	const char* id;

	// API Level 0, Feature Level 0

	/** Perform a search in the provider.
	 *
	 * Providers may offer their own search customization via the returned object
	 * and a custom API.
	 * 
	 * This is a blocking operation which may perform long work if additonal data
	 * is required to perform this action. It should be called asynchronously in
	 * another thread to avoid freezes or stuttering.
	 * 
	 * @param filter The filter to apply to the search, or NULL if all should be
	 *   listed.
	 * @param count Set to the number of plugins matching the filter, or 0 if an
	 *   error occured.
	 * @param error Set to zero unless an error occured. In that case it is set
	 *   to a buffer which contains a C-string, which must be valid until it is
	 *   `bfree` by the caller.
	 * @return An internal object used for further information.
	 */
	void* (*create_search)(const char* filter, size_t& count, const char*& error);
	
	/** Retrieve a single result from the search.
	 *
	 * Only plugin information for currently supported plugins should be 
	 * returned. In the event that a plugin
	 *
	 * This is a blocking operation which may perform long work if additonal data
	 * is required to perform this action. It should be called asynchronously in
	 * another thread to avoid freezes or stuttering.
	 * 
	 * @param search The search created 
	 * @param idx The index of the plugin
	 * @return TODO
	 */
	obs_plugin_information_t (*get_search_result)(void* search, size_t idx, const char*& error);

	/** Destroy a previously created search
	 * 
	 * This is a blocking operation which may perform long work if additonal data
	 * is required to perform this action. It should be called asynchronously in
	 * another thread to avoid freezes or stuttering.
	 * 
	 * @param search The search that should be destroyed.
	 */
	void (*destroy_search)(void* search);

	/** Count the number of update channels this provider supports.
	 * 
	 * Some providers may offer more than a single update channel for plugins, 
	 * enabling plugin authors to offer unstable or even testing versions to
	 * curious users.
	 *
	 * @return The number of update channels available.
	 */
	size_t (*count_channels)();

	/** Retrieve the name of a specific update channel index.
	 *
	 * @param idx Index to look at.
	 * @return A valid C-string if idx is valid, otherwise 0.
	 */
	const char* (*get_channel_name)(size_t idx);
	
	/**
	 * 
	 */
};

typedef obs_plugin_provider* obs_plugin_provider_t;
```

## Plugin Information API
The plugin information API provides minimum information about a plugin, such as the id, name, 

### Structure
```
struct obs_plugin_information {
	/** The API and Feature level that the plugin provider supports.
	 *
	 * Consists of two 32-bit unsigned integers stored with the current CPUs 
	 * endianness, storing the API level in the bits [0...31] and the feature 
	 * level in the bits [32...63].
	 *
	 * The API level increases only when incompatible API changes are made, such
	 * as deleting or moving members. It should be considered unsafe to continue
	 * using the API if a higher or lower API level is found.
	 *
	 * The feature level increases for additions to the current API level and
	 * resets back to zero if a new API level is introduced. It is safe to use a
	 * higher API level than supported, but not a lower one.
	 */
	uint64_t api_feature_level;

	/** The unique id of the plugin.
	 * 
	 * This is provided by the author of the plugin and should ideally be 
	 * identical across providers. The name should ideally be in a reverse-DNS
	 * structure (a.b[.c[.d[.e[...]]]]) and must match the following regular
	 * expression: `[a-zA-Z0-9\-\.]+`.
	 *
	 * May not be set to zero, an empty string, or be deleted before the end of
	 * the lifetime of the provider.
	 */
	const char* id;

	// API Level 0, Feature Level 0

	/** Version of the plugin as "A.B.C.D".
	 *
	 * A C-string with the format "A.B.C.D" where each components matches the
	 * regular expression /[0-9]+/.
	 */
	const char* version;

	/** The provider that provided this plugin.
	 *
	 * Used for checking for updates as well as other information, and as such
	 * must never be 0. Must be valid as long as the provider is loaded.
	 */
	obs_plugin_provider_t provider;


}

typedef obs_plugin_information* obs_plugin_information_t;
```

## Plugin Structure
A plugin will have this structure when installed by a provider:

* `./info.json`
    A JSON file that contains all information about _this_ version of the plugin, such as the version number, dependencies, provider id that provided it, and other useful meta information. This file is automatically created by the provider once the plugin has been installed.
* `./bin/(platform)-(arch)/PLUGIN_ID.EXT`
    * (platform) may be any of: `windows`, `macos`, `linux` (Generic Linux), `ubuntu`, `freebsd`, ...
    * (arch) may be any of: `x86`, `x86_64`, `arm`, `arm64`, ...
    * PLUGIN_NAME.EXT should be replaced with the plugin id and platform extension.
    * Examples:
	    * `./bin/windows-x86/com.xaymar.streamfx.dll`: Windows, x86, 32-bit
		* `./bin/windows-x86_64/com.obsproject.overlay.dll`: Windows, x86, 64-bit
		* `./bin/macos-x86_64/com.example.example.so`: MacOS, x86, 64-bit
		* `./bin/macos-arm64/com.obsproject.vst2.so`: MacOS, ARM, 64-bit
		* `./bin/linux-x86_64/com.xaymar.streamfx.so`: Linux, x86, 64-bit
* `./data/`
   All of the plugin data for all platforms. This should not contain per-platform binaries, those should be placed in `./bin/(platform)-(arch)/` and loaded from there.

### Information JSON
Lines prefixed with `//` are comments and not part of the JSON specification and should be ommitted from the actual file. This is Pseudo-JSON only meant for documentation and specification, and will fail validation.

```
{
	// API and feature level
	"api": 0,
	"api_feature": 0,

	// The idenfitier of the plugin, as a reverse-DNS string. This must match 
	// the regular expression /[a-zA-Z0-9\.\-]+/.
	"id": "com.example.local",
	
	// Version of the plugin in the format: 1.2.3.4
	"version": "1.2.3.4",

	// Localized name for the plugin in the format:
	//   "locale": "name"
	// With "locale" being the same format as OBS locales.
	"name": {
		"en-US": "My Readable Name",
		"de-DE": "Mein übersetzter Name",
		...
	},

	// List of authors that worked on the plugin.
	"authors": [
		"Firstname Lastname Nickname",
		...
	],

	// Dependencies
	"depends": {		
		// Each dependency is named by their unique id.
		"obs": {
			// (Required) Minimum version that can run this plugin, as an inclusive test.
			"min": "1.2.3.4",

			// (Optional) Maximum version that is supported by the plugin, as an inclusive test.
			"max": "2.3.4.5",

			// (Optional) An array of specific versions to exclude that this plugin does not work with.
			"exclude": [ 
				"1.2.3.5",
				"2.3.3.4",
				...
			]
		},
		...
	},
}
```

# Drawbacks
1. It may be possible to escalate privileges from userland if the user ran libOBS with administrator rights. This is technically outside of the scope of this RFC however, as this only addresses managing and providing plugins and their information.

# Additional Information
1. The plugin manager could safely handle load-time crashes and disable plugins that crashed the process using a tiny tracking file.
2. It should be possible to install more than one plugin version at the same time, with the plugin manager automatically picking the latest supported version.

# Likely to be thrown away before Draft is finished

## Plugin Information Metadata
Lines prefixed by // are for documentation and not part of the JSON.

```
{
// Version of the metadata, should currently be set to 1.
	"api": 1,

// Name of the plugin, in the current libOBS locale (if available).
	"name": "name",

// Description of the plugin, in the current libOBS locale (if available).
// This member is optional, and may be ommitted.
	"description": "description",

// Array of supported architectures for this plugin.
	"archs": [
		"x86",
		"x86_64",
		"arm",
		"arm64",
		...
	],

// Array of supported operating systems for this plugin.
	"os": [
		"windows",
		"macos",
		"ubuntu",
		"freebsd",
		...
	],


// Media URLs attached to the plugin as an Array of:
// {
// 	"url": "https://myurl.a.b/file.png",
//  "type": "icon"
// }
	"media": [
		{
			"url"
		},
		...
	],

}
```

## Plugin Information Version Metadata
```
{
// Version of the metadata, should currently be set to 1.
	"api": 1,
	
// Full version description in the A.B.C.D format, where A.B.C.D are numbers from 0 to 65535.
	"version": "A.B.C.D",

// Dependencies of this version.
	"depends": {
		"obs": {
			"min": "A.B.C.D",
			"max": "A.B.C.D"
		},
		...
	}


}
```