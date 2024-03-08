- Start Date: 2020-04-04
- RFC PR: #16

# Summary

Add the ability to stream output audio to a virtual microphone in Windows, Mac and Linux.

# Motivation

OBS is a powerful set of tools to manipulate live video and audio streams that natively supports output to popular web streaming services as well as rendering to a local video file. There are a huge number of people who engage in 1:1 or small-scale streaming using video conferencing software like Zoom or Google Hangouts. Many of these people have similar needs to those of traditional streamers, but for one reason or another cannot switch the video conferencing software they use (social graph, corporate policy). One way to deliver OBS output to these applications is to provide a virtual webcam and microphone in OBS. In conjuction with RFC#15 this will implement a whole suite of software that will allow other applications to use OBS Output as a camera.

Similar functionality: 
- [linux-puseaudio](https://github.com/obsproject/obs-studio/tree/master/plugins/linux-pulseaudio) already implements pulseaudio output for linux which could be used from client applications already (maybe? no really checked the code yet)

# Detailed design

Client/Server IPC between obs-studio and OS HAL (HardwareAbstractionLayer).

- There are already crossplatform audio streaming solutions like pulseaudio.

## Technical considerations

- This RFC is explicitly for an OBS-specific implementation of output from OBS to a virtual microphone.
- A generic middleware solution to this problem is out of scope
- A virtual micrphone needs to be registered with the system. Usually, this'll be done via the OBS installer/updater
- The virtual device/output should be accessible by third party applications that support microphone input

## User UX

- Add a button in the OBS UI below "Start Recording" and above "Studio Mode". In English localization, the button would be labeled "Start Virtual Microphone".
- When you click this button, the current output (either the preview output or the broadcast depending on studio mode) is directed to the Virtual Microphone Device. The button text then toggles to the localized "Stop Virtual Microphone" label.
- When the user clicks into the settings of their video conferencing app, they will see an entry in the list of cameras labeled "OBS Virtual Camera" (no localisation). When they choose this, the OBS output will be fed from this "device" to the app.
- When the user clicks "Stop Virtual Camera", the virtual device no longer receives frames from OBS. It is up to the consuming application what to do in this state.

# Drawbacks

Platform-specific driver code would require maintenance as operating systems (especially in more recent times) tend to limit access (or require user permission) to access/provide to AV hardware. This especially will mean more testing/debugging will be required each time Windows, macOS, and officially supported Linux flavours (currently Ubuntu) provide a major update.

# Alternatives

* Extend Mac & Linux OBS via plugin interface ala Windows-only OBS virtual camera. Clunkier, harder to use especially for novice users, but doesn't add as much complexity to base OBS.

* Pursue a more generic output interface and (encourage someone else to?) build a generic virtual camera to consume it