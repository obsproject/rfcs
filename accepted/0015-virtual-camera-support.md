- Start Date: 2020-03-25
- RFC PR: #15
- Related Github Issue: https://github.com/obsproject/obs-studio/issues/25685

# Summary

Add the ability to render output to a virtual camera device on Linux, Mac, and Windows.

# Motivation

OBS is a powerful set of tools to manipulate live video streams that natively supports output to popular streaming services as well as rendering to a local video file. There are a huge number of people who engage in 1:1 or small-scale streaming using video conferencing software like Zoom or Google Hangouts. Many of these people have similar needs to those of traditional streamers, but for one reason or another cannot switch the video conferencing software they use (social graph, corporate policy). One way to deliver OBS output to these applications is to provide a virtual camera in OBS.

Similar functionality: 
* OBS can be extended by the [OBS-VirtualCam plugin](https://obsproject.com/forum/resources/obs-virtualcam.539/). Currently, only Windows is supported.
* [Wirecast includes virtual camera support](http://www.telestream.net/pdfs/user-guides/Wirecast-8-User-Guide-Windows.pdf) on both Windows and Mac.
* [It was possible](https://github.com/zakk4223/SyphonInject) to inject Syphon functionality into a process that draws to a GL context until some Mac security changes broke this.
* [The Snap Camera](https://snapcamera.snapchat.com) is an example of an application that consumes input from a hardware webcam, processes the video stream, and outputs it in real time as a virtual device that appears in apps like Skype, Hangouts, or Zoom. It supports both Mac & PC.

# Detailed design
- TODO - I don't know anything about OBS internals and so don't know how to design this.
- Platform specific device driver?

## Technical considerations

- TODO

## User UX

- Add a button in the OBS UI below "Start Recording" and above "Studio Mode". In English localization, the button would be labeled "Start Virtual Camera".
- When you click this button, the current output (either the preview output or the broadcast depending on studio mode) is directed to the Virtual Camera Device. The button text then toggles to the localized "Stop Virtual Camera" label.
- When the user clicks into the settings of their video conferencing app, they will see an entry in the list of cameras labeled "OBS Virtual Camera" (or localized equivalent). When they choose this, the OBS output will be fed from this "device" to the app.
- When the user clicks "Stop Virtual Camera", the virtual device no longer receives frames from OBS. It is up to the consuming application what to do in this state.

# Drawbacks

Specializing output to "virtual camera" is more maintenance load for OBS team, more complexity, and presumably, 3x complexity as the approach to create a virtual device probably involves a fair amount of platform-specific driver code. It's a very important case of video streaming with many millions of users, but it's still something of a special case.

# Alternatives

* Extend Mac & Linux OBS via plugin interface ala Windows-only OBS virtual camera. Clunkier, harder to use especially for novice users, but doesn't add as much complexity to base OBS.

* Pursue a more generic output interface and (encourage someone else to?) build a generic virtual camera to consume it. E.g. output via RTSP, NDI and have a "NDI Cam" that has nothing to do with OBS.
