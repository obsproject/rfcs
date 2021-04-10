# Summary

Add built-in backup functionality to OBS Studio.

# Motivation

As of writing OBS does not offer any native option for users to have their settings (profiles and scene collections)
automatically backed up to cloud, local, or network storage. At the moment backing up OBS' entire configuration data
is also not easily possible or intuitively explained for non-technical users.

This RFC is meant to firstly facilitate a discussion to determine a high-level approach to implementing a backup
feature in OBS Studio, before then formulating a specification for the low-level implementation of the chosen approach.

# Detailed changes

To start, I propose the following three approaches for discussion (though more should be suggested and discussed):

 1. Automatic export to user-specified storage location
 2. Integration of third party (cloud) storage providers
 3. Backup infrastructure and account system operated by the OBS Project

Additionally, this RFC will lay out some ideas for how to integrate these backup/restore functionality into the OBS UI.

## Overview of the three suggested variants

### 1. Automatic export

Arguably the easiest approach is to add an option to automatically export/save OBS configuration files to an additional
location when they are written. This could be configured by the user to be a synced folder (e.g. Dropbox or Google Drive)
or a network drive, but could also be simply a different storage drive for redundancy.

Versioned backups could be implemented by allowing backups to be sorted by date or time in the filename or folder path.

### 2. Third party storage providers

For this option OBS would include native support for at least one third party storage provider, either through their
public APIs or GPL-compatible SDKs provided by the platform.

Examples include:

- Dropbox
- Google Drive
- Microsoft OneDrive
- Platforms that support standardized APIs such as WebDAV or S3

Ideally the providers implemented are available to end users directly and already have a large user base among OBS users.
Google Drive or Dropbox would likely be good candidates for the latter case.

In the case of Dropbox or Google we could integrate the providers' native SDK or REST API with OBS in order to upload,
download, and manage backed up files. They also support version histories (at least 30 days) that could be exposed in
OBS to allow for restoring older versions e.g. after accidental changes or deletions.

This is likely the favoured option for feature set and cost, since using the APIs would likely be free to the OBS Project
and users, though at least Google may require a lengthy review process to allow full access to APIs (see [Restricted scopes])

[Restricted scopes]: https://support.google.com/cloud/answer/9110914#restricted-scopes

### 3. OBS Project provided solution

In this case the OBS Project would provide storage and an account system for uploading, downloading, and authenticating backups.
For example this could be implemented by authenticating uploads/downloads to/from an object storage backend that is hosted by a
third party such as Amazon AWS, Backblaze, or OVH.

This approach has a lot of downsides; namely operating and development cost, data security and maintenance requirements.

Due to complexity and cost this likely the least favored approach to backups for everyone involved.

## UI Changes

The configuration for OBS backups would be handled by a new tab in the settings titled "Backup".
This would contain the options to authenticate and configure the chosen backup provider and may contain statistics and
information about the number, size, and up-to-date ness of backups. Options shall include frequency, maximum size,
and whether the backup system shall be disabled while an output is active.

Restoring backups should be possible in at least two ways: full restore and individual file restore.

For full restore the controls shall be present in the backup settings tab and allow to restore all profiles and collections
at once based on the latest uploaded full backup. For individual restore the option should be part of the "Profile" and
"Scene Collections" dropdown, named "Import from Backup", and be placed adjacent to the "Import" and "Export" features.
These would invoke a Qt dialogue presenting a list of backed up collections/profiles, optionally allowing the user to restore
a certain revision if the storage backend supports this functionality.

## Further considerations

As proposed the backup functionality would *not* back up assets used as part of a scene collection.
Backing up assets may however be desirable in order to make OBS backups truly portable across machines.

Adding features such as proposed in [OBS PR #2233] to more easily add missing assets can alleviate this however, as long 
as the assets are stored in some synchronized storage system (e.g. the `%USERPROFILE%/Dropbox` directory) by the user.

[OBS PR #2233]: https://github.com/obsproject/obs-studio/pull/2233

# Additional Information

## Possible cloud providers' third party integration documentation

- Google Drive: https://developers.google.com/drive
- Dropbox: https://www.dropbox.com/developers/documentation
- OneDrive: https://developer.microsoft.com/en-us/onedrive
