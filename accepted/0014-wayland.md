- Start Date: 2020-03-09
- RFC PR: #14
- Mantis Issue: N/A

# Summary

Add native Wayland support for OBS Studio

# Motivation

Wayland is a big and widespread alternative to the X11 protocol. Most desktop
environments on Linux support Wayland now, and it is the default on GNOME.
There are also many Wayland-only window managers being created every day.

OBS Studio has X11-specific code, which makes it incompatible with Wayland.

# Detailed design

This RFC covers both making OBS Studio be able to run as a native Wayland
application, and also adding screen and window captures that work on Wayland
compositors.

## Native Client (✓ done)

I hereby propose a 3-step approach to achieve native Wayland integration:

 1. Introduce an EGL-X11 renderer to `libobs-opengl` (✓ done)
 2. Add the concept of a Wayland platform to `libobs` (✓ done)
 3. Introduce an EGL-Wayland renderer to `libobs-opengl` (✓ done)

Optionally, a 4th step may be:

 4. Add an experimental checkbox to the Configuration dialog to enable EGL;

A detailed explanation of each step follows below.

### EGL-X11 (✓ done)

This step involves adding an abstraction layer to the windowing system, renaming
`libobs-opengl/gl-x11.c` to `libobs-opengl/gl-glx.c`, and introducing a new
`libobs-opengl/gl-egl-x11.c` file.

The `glad` dependency would also be updated to have the necessary EGL files.

The abstraction layer is composed of `libobs-opengl/gl-nix.{c|h}`. This is where
the exported functions (i.e. everything included in `struct gs_exports`) are
implemented. All these function implementations, however, will look like this:

```c
extern void gl_foo_bar(...)
{
	gl_winsys->foo_bar(...);
}
```

That's because `gl_winsys` is selected runtime, and may be either the GLX winsys,
or the EGL/X11 winsys.

### Wayland Platform (✓ done)

The Wayland Platform step introduces a way to detect at runtime which platform
OBS Studio is running on. Initially, only Unix implementations will set and
actually use it.

The header file would look like:

**libobs/obs-nix-platform.h**

```c
enum obs_nix_platform_type {
	OBS_NIX_PLATFORM_X11_GLX,
	OBS_NIX_PLATFORM_X11_GLX,
	OBS_NIX_PLATFORM_WAYLAND,
};

EXPORT void obs_set_nix_platform(enum obs_nix_platform_type platform);
EXPORT enum obs_nix_platform_type obs_get_nix_platformobs_get_nix_platform(void);

EXPORT void obs_set_nix_platform_display(void *display);
EXPORT void *obs_get_nix_platform_display(void);
```

### EGL-Wayland (✓ done)

The 3rd and last step is introducing the `libobs-opengl/gl-egl-wayland.{c|h}`
files, and making `gl-nix` retrieve the EGL/Wayland winsys. This is the only
winsys that can be used when running under Wayland.

## Screen & Window Capture

The process for capturing screen and window contents on Wayland compositors
is different than on Xorg. While X11 gives applications the ability to spy on
anything, including other applications, the entire compositor, Wayland proposes
a much stricter security model.

### Portals

On Wayland, the general way of requesting the compositor for sharing the screen
or window contents is through the [ScreenCast portal](screencast-portal).

Portals are D-Bus interfaces that applications use to interact with the desktop.
The ScreenCast portal specifically provides applications the necessary tools to
ask the desktop environment to share the contents of a window or a monitor.

Known to implement the ScreenCast portal is [GNOME](gnome-screencast),
[KDE](kde-screencast), and [wlroots](wlroots-screencast). These three should
cover the majority of Wayland compositors in existence today.

### PipeWire

The ScreenCast portal is written based on [PipeWire](pipewire) to work. PipeWire
is a service that provides a robust and performant media sharing mechanism for
video and audio.

There already is an out-of-tree plugin, [obs-xdg-portal](obs-xdg-portal), that
implements this. It can serve as a basis for either a new in-tree plugin, or
be incorporated as part of the `linux-capture` plugin.

# How We Teach This

N/A, this simply changes the current behaviour. As long as the patch notes
properly describe the changes, we should be fine.

# Drawbacks

This doesn't change the default behavior of OBS Studio, which reduces the
surface area for regressions.

# Alternatives

Not supporting Wayland?

# Unresolved questions

 * Should the capture code be a new in-tree plugin, or incorporated into `linux-capture`?



[gnome-screencast]: https://github.com/flatpak/xdg-desktop-portal-gtk/blob/master/src/screencast.c
[kde-screencast]: https://github.com/KDE/xdg-desktop-portal-kde/blob/master/src/screencast.cpp
[obs-xdg-portal]: https://gitlab.gnome.org/feaneron/obs-xdg-portal/
[pipewire]: https://pipewire.org/
[screencast-portal]: https://github.com/flatpak/xdg-desktop-portal/blob/master/data/org.freedesktop.impl.portal.ScreenCast.xml
[wlroots-screencast]: https://github.com/emersion/xdg-desktop-portal-wlr/blob/master/src/screencast/screencast.c
