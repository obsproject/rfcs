# Summary
Allow users to install and load plugins without requiring administrator privileges.

# Motivation
The requirement of administrator rights to add plugins confuses users for strange reasons.

# Drawbacks
- Introduces a security vulnerability on Windows, where it is common to run OBS as adminsitrator for the improved GPU scheduling.

# Design
OBS Studio should be able to load both system-wide and user-local plugins on every supported platform, according to that platforms specifications. This allows users to customize their own OBS Studio installation without requiring an administrator to be present for every change.

## Windows
Plugins should be loaded from the system-wide directory, as well as the user-local directory `%LOCALAPPDATA%\obs-studio\plugins` (if it exists). Plugins should not be considered roaming information, as there is no guarantee that the user profile is not transferred to a machine using 32-bit or even ARM - or even the same OS.

## Linux (XDG)
The [XDG Specification](https://specifications.freedesktop.org/basedir-spec/basedir-spec-latest.html) does not explicitly say where plugins should be stored, but going by contextual information it should be loaded from `$XDG_DATA_HOME/obs-studio/plugins` as well as the system-wide directory. If $XDG_DATA_HOME is not specified, `$HOME/.local/share/obs-studio/plugins` should be used instead.

## MacOS
This OS appears to already support user-local and system-wide plugin installations.

## Load Priority
If possible, the user-local directory should be loaded after the system-wide directory.

# Migration
No migration is necessary in order to support this.
