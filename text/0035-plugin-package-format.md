# Summary
A unified way for plugins to be packaged for all platforms.

# Motivation
Currently all platforms have their own way of shipping plugins, some even being different by the way they are installed (system-wide vs local-user), which results in higher difficulty when shipping cross-platform plugins. The current behavior also prevents some much needed future features from even working correctly.

# Drawbacks
Potentially more complicated than current logic.

# Design
All plugins should be shipped and loaded in the following format:

- `/<name>/data`
    The plugins custom data, such as localized text and images.
- `/<name>/bin[/<obs version>]/<platform>/[.../]<name>[.<ext>]`
    Binaries for a specific platform. Platforms may choose to do additional parsing of the path in order to support version selection, architectures, and bitness.
- **or** `/<name>/bin/<name>[.<ext>]`
    Binaries for an intelligent plugin manager which can intelligently pick out specific variants of the binaries to actually install.

## Keywords
### Keyword: `<name>`
The unique name of the plugin which must match the actual module name, excluding the file extension. This name can be anything, as long as it is valid for the platform and/or container the plugin is in.

### Optional Keyword: `<obs version>`
Minimum OBS Studio version supported by the binaries in the directory in the format `A[.B[.C[.D]]]`, where each component is a positive integer number. If left out, assumes that all OBS versions are compatible.

### Optional Keyword: `<ext>`
A platform specific extension to denote dynamic libaries. A generic extension, such as `bin`, `obs` or similar may also be supported, depending on the final implementation, but this RFC does not specify that anything but the platform native extension for dynamic libraries. On platforms where no such extension is required, the dot prefixing the extension is also removed.

### Keyword: `<platform>`
The platform for which the binary is made for. Platforms may opt for additional parsing of the `[.../]` part in order to filter by platform version, architecture or bitness. By default the following platforms are available:

#### `windows`
The Windows platform supports the additional keywords `<version>`, `<arch>` and `<bits>`, which should be written as: 

```
/<name>/bin[/<obs version>]/windows[/<version>]/<arch>-<bits>/<name>[.<ext>]
```

- `<version>` is optional and defines the system version (i.e. 10.0.19042).
- `<arch>` is required.
- `<bits>` is required.

This layout allows us to (optionally) specify a minimum platform version, a required architecture, and a required bitness. For example, with this we can filter for Windows 10 20H2 x86 64-bits with the following text: `/<name>/bin/windows/10.0.19042/x86-64/<name>[.<ext>]`, and as such prevent the plugin from loading on unsupported platforms automatically.

#### `mac` or `macos`
The Mac/MacOS platform supports the additional keywords `<version>`, `<arch>` and `<bits>`, which should be written as:

```
/<name>/bin[/<obs version>]/mac[/<version>]/<arch>-<bits>/<name>[.<ext>]
```

- `<version>` is optional and defines the system version (i.e. 10.13).
- `<arch>` is required.
- `<bits>` is required.

This layout allows us to (optionally) specify a minimum platform version, an required architecture, and a required bitness. We can easily filter for specific Apple devices, and require certain OS versions for plugins, or even have M1 or X86 exclusive plugins.

#### `linux`
The Linux platform supports the additional keywords `<distro>`, `<version>` `<arch>` and `<bits>`, which should be written as:

```
/<name>/bin[/<obs version>]/linux[/<distro>[-<version>]]/<arch>-<bits>/<name>[.<ext>]
```

- `<distro>` is optional, and specific distributions may have a fallback to other values if no specific version is present.
- `<version>` is optional, and describes the distro-specific version, for example:  
    **On Ubuntu:** 18, 18.04, 19, 19.10, 20, 20.04, ...
    **On Debian:** 9, 10, 11, ...  
    **On FreeBSD:** 9, 9.0, 9.1, 10, ...
- `<arch>` is required.
- `<bits>` is required.

### Keyword: `<version>`
Specifies a minimum version in the format `A[.B[.C[.D]]]`, where each component is a positive integer number. Each platform can parse this information how they want, and may specify if this keyword is optional or not.

### Keyword: `<distro>`
Linux exclusive keyword to specify the exact distribution on which the plugin can run, such as `ubuntu`, `debian`, `freebsd`, etc. If a specific distro is missing, a fallback may be used to load binaries from another distro known to be compatible (for example, `ubuntu` may load `debian` and vice versa).

### Keyword: `<arch>`
The architecture keyword is used to denote the architecture, with these values being available:

- `any`
    Runs on any architecture, see FatELF on Linux or Universal Binary on MacOS
- `x86` or `X86`
    Intel/AMD x86 based system.
- `arm` or `ARM`
    ARM based system, such as a Windows ARM laptop, or a Apple MacBook Pro with M1.

### Keyword: `<bits>`
An integer value describing the bitness of the underlying architecture, used for pointers and similar critical elementsFinally, the bitness of the system. This is an integer of any value, but the common numbers are:

- `0` (aka `any`, as it fails translation into an integer)
    Any bitness, see FatELF on Linux or Universal Binary on MacOS.
- `32`
    32-bits
- `64`
    64-bits

Future bitness support is easily added by simply just increasing the number, so if we ever end up with 128-bits, just use 128.

## Behavior when loading Plugins
1. Attempt to load any built-in plugins of either old or new style shipping.
2. Attempt to load any new style plugins.
    - If the loading fails, log it and continue with the next plugin.
    - Otherwise, allow the plugin to register additional names to be excluded from legacy/old style loading.
3. Attempt to load any old style (legacy) plugins that do not match any registered load exemptions or known new style plugins.
    - If the loading fails, log it and continue with the next plugin.

### Preferred Architecture and Bitness on Multi-Arch Platforms
Loading of binaries should prefer the Host architecture and bitness first, before attempting to load anything else. This allows a plugin to perform specific optimizations for `arm-64` and `x86-64` which may not be possible in a fat/universal binary. For example if the host is `arm` and `32`, the first test should look for those two, then look for `any` and `32`, then look for `arm` and `any`, and finally look for `any` and `any`. See the following list for a better and longer example:

- Host is x86-32:
    1. Check if `/<name>/bin/<platform>/x86-32/<name>.<ext>` exists, if yes attempt to load it.
    2. Check if `/<name>/bin/<platform>/any-32/<name>.<ext>` exists, if yes attempt to load it.
    3. Check if `/<name>/bin/<platform>/x86-any/<name>.<ext>` exists, if yes attempt to load it.
    4. Check if `/<name>/bin/<platform>/any-any/<name>.<ext>` exists, if yes attempt to load it.
- Host is x86-64
    1. Check if `/<name>/bin/<platform>/x86-64/<name>.<ext>` exists, if yes attempt to load it.
    2. Check if `/<name>/bin/<platform>/any-64/<name>.<ext>` exists, if yes attempt to load it.
    3. Check if `/<name>/bin/<platform>/x86-any/<name>.<ext>` exists, if yes attempt to load it.
    4. Check if `/<name>/bin/<platform>/any-any/<name>.<ext>` exists, if yes attempt to load it.
- Host is arm-32:
    1. Check if `/<name>/bin/<platform>/arm-32/<name>.<ext>` exists, if yes attempt to load it.
    2. Check if `/<name>/bin/<platform>/any-32/<name>.<ext>` exists, if yes attempt to load it.
    3. Check if `/<name>/bin/<platform>/arm-any/<name>.<ext>` exists, if yes attempt to load it.
    4. Check if `/<name>/bin/<platform>/any-any/<name>.<ext>` exists, if yes attempt to load it.
- Host is arm-64:
    1. Check if `/<name>/bin/<platform>/arm-64/<name>.<ext>` exists, if yes attempt to load it.
    2. Check if `/<name>/bin/<platform>/any-64/<name>.<ext>` exists, if yes attempt to load it.
    3. Check if `/<name>/bin/<platform>/arm-any/<name>.<ext>` exists, if yes attempt to load it.
    4. Check if `/<name>/bin/<platform>/any-any/<name>.<ext>` exists, if yes attempt to load it.

## Examples
### MacOS 10.15 plugin on a 10.13 machine (and vice versa)
```
/test/data/license.txt
/test/bin/mac/10.15/any-64/test.so
/test/bin/mac/10.13/any-64/test.so
```

- On the 10.13 machine:
    1. Compare 10.13 against 10.15, this test should fail.
    2. Compare 10.13 against 10.13, this test should succeed, so we load `/test/bin/mac/10.13/any-64/test.so`
- On the 10.15 machine:
    1. Compare 10.15 against 10.15, this test should succeed, so we load `/test/bin/mac/10.15/any-64/test.so`

### Loading a Windows 10 plugin on Windows 8.1
```
/test/data/license.txt
/test/bin/windows/10.0.19042/x86-64/test.dll
/test/bin/windows/10/x86-64/test.dll
/test/bin/windows/x86-64/test.dll
```

1. We compare the current version against `10.0.19042`, which should not match.
2. We compare the current version against `10`, which should not match.
3. We compare the current version against ``, which should match, so we load `/test/bin/windows/x86-64/test.dll`.

### Ubuntu 20.04 Plugin with "generic" distro fallback
```
/test/data/license.txt
/test/bin/linux/ubuntu/20.04/x86-64/test.so
/test/bin/linux/ubuntu/x86-64/test.so
```

- Host is Ubuntu 18.04:
    1. Compare `18.04` against `20.04`, this test should fail.
    2. Compare `18.04` against ``, this test should succeed, so we load `/test/bin/linux/ubuntu/x86-64/test.so`.
- Host is Ubuntu 19.10:
    1. Compare `19.10` against `20.04`, this test should fail.
    2. Compare `19.10` against ``, this test should succeed, so we load `/test/bin/linux/ubuntu/x86-64/test.so`.
- Host is Ubuntu 20.04:
    1. Compare `20.04` against `20.04`, this test should succeed, so we load `/test/bin/linux/ubuntu/20.04/x86-64/test.so`.
