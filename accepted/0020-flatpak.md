- Start Date: 2020-04-18
- RFC PR: #21
- Mantis Issue: N/A

# Summary

Flatpak is an app packaging and distribution mechanism widely available on Linux
distributions. To some distributions, Flatpak is part of the default experience.
Flatpak allows tighter control over the environment applications run in, how they
are packaged and installed, and also isolates the application from the host
system.

The goal is add support for Flatpak, both as a development platform, and also as
a distribution platform.

# Motivation

As a complex multi-platform project, OBS Studio has a wide surface for breakage.
Bad packaging can be a real problem, given that many Linux distributions package
OBS Studio in slightly different and incompatible ways.

By supporting Flatpak, OBS Studio benefits from having a tight control over the
execution environment, and the packaging of plugins and dependencies. This will
reduce the number of moving part when running OBS Studio, which allows much easier
reproduction of bugs and, consequently, fixing them.

## Internals

There are no code changes involved in adding Flatpak support. It is simply a
matter of adding a new file. This file is JSON formatted file containing the
dependencies, permissions, and the platform that OBS Studio depends on.

# How We Teach This

Because this is an addition to the platform, users shouldn't be required to learn
about Flatpak. For developers, little will change.

# Drawbacks

No known drawbacks.

# Additional Information

Supporting Flatpak does not prevent OBS Studio from supporting other distribution
mechanisms, nor will affect Linux distributions that package OBS Studio manually.

# Unresolved questions

 * Should OBS Studio use Flatpak as part of the CI process?
 * Should Flatpak be part of the release process?
