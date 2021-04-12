# Summary

This RFC proposes distribution of OBS on Valve's Steam platform as well as considerations for OBS distribution on other third-party
platforms in the future.

# Motivation

Over the years users have asked us to provide a Steam version of OBS for a few reasons:

* Convenience (automatic updates, one-click install)
* Non-Admin Users can install it with the Steam Service taking care of elevated operations
* Other commonly used utilities (e.g. ShareX, Precision, Video editors) already exist on Steam,
  making it a "One Stop Shop" for many people's gaming content creation needs

Additionally, certain Steamworks feature can allow us to offer a few optional features,
including some that currently are on the mid- to long-term roadmap for OBS, right away:

* Steam Cloud for syncing/backing up OBS settings and scene collections
* Opt-in branches for release candidates (or older builds)

It should be noted that we want to avoid vendor lock-in and thus refrain from adding Steam-specific code to OBS.
All of the features listed above are usable without having to add any code to OBS whatsoever.

The deployment process to Steam appears to be nearly entirely automatable with our current CI/CD solution (GitHub Actions),
only requiring human input to approve non-beta updates.

This makes providing a Steam distribution of *just* the latest Windows Release a potentially very low overhead affair.

Additionally, the lessons learned may prove useful for other distribution platforms that have been proposed (e.g. Microsoft Store).

# Detailed changes

## Changes to OBS

### Accommodating cloud saves

For the most basic implementation of Steam save backups ([Steam Auto-Cloud]) **no changes to OBS are required**. After configuration through
the Steamworks web interface the Steam client will automatically back up selected files and folders in `%APPDATA%\obs-studio` once OBS exits.

However, since "cloud saves" or "cloud backups" offered by platforms may not be specially encrypted we may also consider changes
to OBS to prevent credentials such as stream keys or OAuth tokens from being backed up to a third party service.

While those credentials are intentionally using restricted scopes, we may still opt to exclude OBS profiles (not scene collections) from being
backed up entirely in the short-term, if we do not believe that we can sufficiently educate users or make inclusion of profiles an opt-in feature.

In the mid- to long-term we might want to investigate other methods of storing those credentials, be it as separate files that can be
excluded from the backup (e.g. `oauth.json`) or by saving them in the operating system's key store instead of OBS configuration files.
Though those are things we may not work out until the platform-independent cloud backup feature's development and are not required for
the implementation of a Steam distribution of OBS Studio.

[Steam Auto-Cloud]: https://partner.steamgames.com/doc/features/cloud#steam_auto-cloud

## Steam specific work

In order to publish to Steam we need a Steamworks Partner account, this account should be owned by the OBS Project rather than an individual.

Once the OBS publisher account is set up other Steam user accounts (e.g. maintainers, people responsible for CI/Steam distribution, and build bots)
can be given (limited) access to the Steamworks Partner portal and API to manage OBS' distribution, Steam Store and Community presence.

We cannot use "Steam Direct" for publishing software and will have to apply directly to Valve (see "Accepted types of content" in the [Steam Direct FAQ]).
The fees are not public, but given the presence of other FOSS projects on Steam, and Steam Direct fees being USD 100 it is likely not cost-prohibitive.

[Steam Direct FAQ]: https://partner.steamgames.com/steamdirect

### Install scripts, configuration, and redistributables

Features such as Steam Cloud and the build process itself will require writing configuration files. These do not contain any sensitive information
and should be published together with other configuration files and scripts in a public repo on the obsproject GitHub organization (e.g. `obs-steam`).
Additionally, some configuration has to be done via the Steamworks web interface which may also be documented in said repository..

Documentation for build configuration and uploading: https://partner.steamgames.com/doc/sdk/uploading

Steam allows us to handle parts that the OBS installer is doing via their install scripts,
this includes creating registry entries such as the Vulkan Layer ones in HKLM and cleaning up on uninstall.

Windows 10 or later should not require installing redistributables on OBS install, should we decide to support Windows 7 and 8.1
then Steam allows us to only install those older OS versions. Since Windows 7 is EOL and 8.1 has a dwindling market share we might
consider to simply exclude those operating systems from the Steam distribution.

Finally, creating the Vulkan Layer files in `%PROGRAMDATA%` should probably be left to OBS, this process also does not
require elevation but specific permissions need to be set on the directory to allow for UWP capture to work.
Cleaning up those files on removal can be handles by the uninstall script.

Documentation for install scripts: https://partner.steamgames.com/doc/sdk/installscripts

### GitHub Actions and Steam branches

Uploading a build on Steam is fairly straightforward and can be done with a one-line `steamcmd` command.
For uploading we would use a dedicated "build bot" account with very limited access and with its credentials
being protected using GitHub Action's secure variables feature (see [Building Depots] and [Steam Build Account]).

The GitHub Action for uploading OBS releases should be separate from the current CI and run on a new GitHub release
being published rather than pushes. That Action would download the portable zip, add files relevant to Steam
(e.g. the disable-updater sentinel), and finally prepare and upload that build to the Steam Master Content Server.

This Action may also be run when releases marked as "Pre-Release" are published (e.g. Release Candidates), those would
then be uploaded to a "release-candidate" branch on Steam that users can opt-in to use. Older major releases may also
be kept as separate branches (similar to some games) to allow for downgrading in case there are issues with a new release.

[Building Depots]: https://partner.steamgames.com/doc/sdk/uploading#Building_Depots
[Steam Build Account]: https://partner.steamgames.com/doc/sdk/uploading#Build_Account

# Drawbacks

If the effort required to maintain the Steam distribution after the initial setup are as low as assumed,
there would be practically no drawbacks to having it as an option for OBS developers or users.
Steam does generally not restrict applications' access to the system so issues related to sandboxing or other
restrictive measures are not to be expected.

# Additional Information

## Links

- Steamworks documentation: https://partner.steamgames.com/doc/home
- Form for distributing Non-Game Software: https://help.steampowered.com/en/wizard/HelpWithPublishing?issueid=925
