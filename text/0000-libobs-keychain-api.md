# Summary

This RFC proposes the addition of OS keychain APIs to the libobs platform utilities to faciliate secure storage of credentials.

# Motivation

While the credentials stored by OBS in profiles are not of severe sensitivity due to limited scope, as a modern desktop application we should still do our best to store them securely.

# Detailed Changes

This RFC proposes the addition of the following libobs API methods:

- `bool os_keychain_available(void)`
- `bool os_keychain_save(const char *label, const char *key, const char *data)`
- `bool os_keychain_load(const char *label, const char *key, char **data)`
- `bool os_keychain_delete(const char *label, const char *key)`

As the API requires OS-specific implementations it shall become part of the `util/platform.h` header.

**Implementation notes:**

Operating-system backends:
- Windows: `wincred.h` / `Cred*` Win32 API functions
- macOS: Keychain Services API
- Linux: `libsecret`

The `label` is a user-visible component that may be displayed in keychain management applications as the label or name of an entry. It must be specified, ASCII only, and must be human-readable.  
In most cases it should identify the component and contents, e.g. for OAuth credentials stored by the OBS Studio UI this could be "OBS Studio OAuth Credentials".  
An entry can only be loaded or deleted if the label and key match the ones specified when saving.

`os_keychain_available()` will always return `true` on Windows and macOS, on Linux it will return `true` if libobs was compiled with libsecret support, but does not indicate whether a keychain backend is available.

**Usage examples:**
Saving:
```c
const char *label = "OBS Studio Stream Keys";
const char *key = "StreamKey_abcef";
const char *value = "mystreamkey";
bool success = os_keychain_save(label, key, value);
```

Loading:
```c
const char *label = "OBS Studio Stream Keys";
const char *key = "StreamKey_abcef";
char *value = NULL;
bool success = os_keychain_load(label, key, &value);

... do stuff ...

if (value)
    bfree(value);
```

Deletion:
```c
const char *label = "OBS Studio Stream Keys";
const char *key = "StreamKey_abcef";
bool success = os_keychain_delete(label, key);
```

# Drawbacks

Accessing keychain information can reduce portability of profiles by not storing information in config files. OBS Studio may opt out of using a keychain if it is running in portable mode.

In some cases accessing the keychain may require user confirmation, the API may block in such cases, which has to be accounted for by API users.

If no keychain is available API users will have to fall back to storing data otherwise (e.g. obs data or config files).

If keychain operations fail API users may have to also fall back to an insecure storage method or warn the user of faiure.

# Additional Information

- Windows credential store documentation: https://learn.microsoft.com/en-us/windows/win32/api/wincred/
- macOS keychain documentation: https://developer.apple.com/documentation/security/keychain_services?language=objc
- Linux/GNOME libsecret documentation: https://gnome.pages.gitlab.gnome.org/libsecret/
- Flatpak "Secret" Portal: https://docs.flatpak.org/en/latest/portal-api-reference.html#gdbus-org.freedesktop.portal.Secret
- Pull request: https://github.com/obsproject/obs-studio/pull/9122
