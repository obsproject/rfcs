# Summary

This RFC proposes general changes to OBS that will allow it to be more easily packaged for distribution on third-party platforms,
starting with Valve's Steam software distribution platform.

# Motivation

Over the years users have asked us to provide a Steam version of OBS for a few reasons:

* Convenience (automatic updates, one-click install)
* Non-Admin Users can install it with the Steam Service taking care of elevated operations
* Other commonly used utilities (e.g. ShareX, Precision, Video editors) already exist on Steam,
  making it a "One Stop Shop" for many people's gaming content creation needs

Additionally, certain Steamworks products can allow us to implement a few optional features,
including some that currently are on the mid- to long-term roadmap for OBS, right away:

* Steam Cloud for syncing/backing up OBS settings and scene collections
* Beta branches and nightlies with automatic updates
* Steam Linux Runtime for widely compatible first party Linux builds
* (potentially) Free DLC packages as a way to offer one-click installation for popular plugins

It should be noted that we want to avoid vendor lock-in and thus refrain from adding Steam-specific code to OBS.
All of the features listed above are usable without having to do that however.

The deployment process to Steam appears to be nearly entirely automatable with our current CI/CD solution (GitHub Actions),
only requiring human input to approve non-beta updates (nightlies/beta branches can be directly published to users).

This makes providing a Steam distribution of *just* the latest Windows Release a potentially very low overhead affair.

The aforementioned Linux distribution is not part of the implementation of this RFC, but my be an option for
distributing first-party Linux builds running on a wide variety of systems (see [Future work]).

Additionally, the lessons learned may prove useful for other distribution platforms that have been proposed (e.g. Microsoft Store).

# Detailed changes

## Changes to OBS

### Disabling the updater

**This has been implemented as of https://github.com/obsproject/obs-studio/commit/30862d75ae5c8ed6d63b6e6aa4139218f48d19e0**

One of the changes to accommodate Steam (and other third party platforms that handle updates) is an option to disable the OBS internal updater.

This should be an option included in normal release builds to avoid having to build separate binaries for Steam/other platforms and
to keep the overhead of maintaining these distributions managable.

I suggest an implementation similar to the portable mode; offering a `--disable-updater` command line switch as well
as a `/disable_updater(.txt)` sentinel file that can be included in distributions.

Note that while Steam allows disabling automatic updates by the publisher (mostly intended for MMOs), this appears to be a global
flag that would also affect additional branches or future Linux builds that do no have an integrated update mechanism.

### Accommodating cloud saves

Generally we do not have to change OBS for the purposes of allowing cloud backups, for those we simply need to inform the platform
about the directories and file types that should be included for backup.

However, since "cloud saves" or "cloud backups" offered by platforms may not be specially encrypted we may also consider changes
to OBS to prevent credentials such as stream keys or OAuth tokens from being backed up to a third party service in an unsecured manner.

While those credentials are intentionally using restricted scopes, we may still opt to exclude OBS profiles from being backed up
entirely in the short-term if we do not believe that we can sufficiently educate users or make inclusion of profiles an opt-in feature.

In the mid- to long-term we might want to investigate other methods of storing those credentials, be it as separate files that can be
excluded from the backup (e.g. `oauth.json`) or by saving them in the operating system's key store instead of OBS configuration files.

The Steamworks documentation on the "Steam Auto-Cloud" can be found here: https://partner.steamgames.com/doc/features/cloud#steam_auto-cloud

## Steam specific work

In order to publish to Steam we need a Steamworks Partner account, this account should be owned by the OBS Project rather than an individual.

Once the OBS publisher account is set up other Steam user accounts (e.g. maintainers, people responsible for CI/Steam distribution, or build bots)
can be given (limited) access to the Steamworks Partner portal and API to manage OBS' distribution, Steam Store and Community presence.

We cannot use "Steam Direct" for publishing software and will have to apply directly to Valve (see "Accepted types of content" in the [Steam Direct FAQ]).
The fees are not public, but given the presence of other GPL projects on Steam and Steam Direct fees being USD 100 it is likely not cost-prohibitive.

[Steam Direct FAQ]: https://partner.steamgames.com/steamdirect

### Install scripts, configuration, and redistributables

Features such as Steam Cloud and the build process itself will require writing configuration files. These do not contain any sensitive
information and should be published together with other configs/scripts in a public `obs-steam` repo on the obsproject GitHub organization.
Additionally some configuration has to be done via the Steamworks web interface.

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

### GitHub Actions/CI

Uploading a build on Steam is fairly straightforward and can be done with a one-line `steamcmd` command.
For uploading we would use a dedicated "build bot" account with very limited access and with its credentials
being protected using GitHub Action's secure variables feature (see [Building Depots] and [Steam Build Account]).

The GitHub Action for uploading OBS releases should be separate from the current CI and run on a new GitHub release
being published rather than pushes. That Action would download the portable zip, add files relevant to Steam
(e.g. the disable-updater sentinel), and finally prepare and upload that build to the Steam Master Content Server.

We may also investigate providing unsigned nightlies by uploading builds to Steam directly from our current CI workflow,
but we will have to check with Valve if uploading builds with that frequency would be acceptable.

[Building Depots]: https://partner.steamgames.com/doc/sdk/uploading#Building_Depots
[Steam Build Account]: https://partner.steamgames.com/doc/sdk/uploading#Build_Account

# Drawbacks

If the above effort required to maintain the Steam distribution after the initial setup are as low as assumed,
there would be practically no drawbacks to having it as an option for OBS developers or users.
Steam does generally not restrict applications on Windows or Linux so issues related to sandboxing or other
restrictive measures are not to be expected.

Finally, we likely cannot use the Steamworks SDK directly within a GPL application ([SDK Documentation about that topic]).
This limits the Steam features we can use to those that work without SDK integration unless we were to invest time into a
solution that is GPL-compatible.

However, as initially mentioned we want to avoid vendor lock-in, so while certain features could offer unique and interesting
features for OBS (e.g. Steam Workshop) we should forgoe those in favor of native and platform-independent implementations in the future.

[SDK Documentation about that topic]: https://partner.steamgames.com/doc/sdk/uploading/distributing_opensource

# Additional Information

## Future work
[Future work]: #future-work

As mentioned initially, Steam offers something called the "Steam Linux Runtime", which is a fixed and stable Linux environment that can be used
to deploy an application to almost any Linux distribution that runs Steam. We may want to investigate using it for providing first party OBS Studio
builds for Linux if the maintenance work required is managable. In case the Flatpak RFC is also accepted, we may still consider it as a method of
distribution that will be more tailored to video game players that have Steam, but not Flatpak, installed.
For instance: the most popular Linux distribution for Steam users are Ubuntu 20.04 and 18.04, which do not include or offer native packages for Flatpak.

## Links

- Steamworks documentation: https://partner.steamgames.com/doc/home
- Form for distributing Non-Game Software: https://help.steampowered.com/en/wizard/HelpWithPublishing?issueid=925
- FOSDEM presentation about the Steam Linux Runtime: https://youtu.be/KrbWbBYAolo
