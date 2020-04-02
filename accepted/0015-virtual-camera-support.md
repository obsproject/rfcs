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

- Platform specific plugins shall be created that will be added as part of the main obs build.
- Each plugin will start a server that will be responsible of transmitting the frame data to a client process.
- A virtualcam "Driver" (client process) will be developed for each platform to consume data incomming from this plugins.
- Each virtual driver shall be part of the installer with a checkbox to install the virtualcam driver

## Cross-platform Interfaces

We can define domain interfaces in order to maintain uniformity in the platform-specific plugins (maybe someday they will not be platform specific).

- **OBSVirtualCamServer**: Interface that defines the set of methods to be implemented by a OBSVirtualCam Server in order to send and receive frame data (and other metadata) to the client processes.
- **OBSVirtualCamClient**: Interface that defines the set of methods to be implemented by a OBSVirtualCam Client in order to register with a server(obs-studio) and receive frame data.

## Platform specific implementations

Each platform specific implementation will be separated into 2 cmake projects (subprojects?).
- **<Platform>OBSVirtualCam OBS Plugin**: Platform speficic obs plugin module that will be loaded by obs-studio in order to expose the virtual cam OBS Output and its controls. This will implement **OBSVirtualCamServer** and interfaces
- **<Platform>OBSVirtualCam OBS Driver**: Platform specific driver code that will register the camera with the OS and will register with the **OBSVirtualCamServer** in order to receive data. If no connection to the server is available it should still register and render an static frame and keep waiting for the **OBSVirtualCamServer**. This will implement **OBSVirtualCamClient** interfaces.

### Windows

#### WindowsOBSVirtualCam OBS Module
#### WindowsOBSVirtualCam Driver

### Mac

#### MacOBSVirtualCam OBS Module
#### MacOBSVirtualCam Drivers

## Technical considerations

- This RFC is explicitly for an OBS-specific implementation of output from OBS to a virtual camera.
- A generic middleware solution to this problem is out of scope at the moment
- A virtual camera needs to be registered with the system. Usually, this'll be done via the OBS installer/updater
- The virtual device/output should be accessible by third party applications that support webcams/capture cards

## User UX

- When installing OBS user should be asked if he would like to install the platform specific driver to enable virtual camera output on the system
- Initial configuration should require very little input from the user, if any. Resolution and framerate of the camera will be pre-defined by the Video settings of OBS, and no effects outside those already provided by OBS will be specially exposed for this output.
- Add a button in the OBS UI below "Start Recording" and above "Studio Mode". In English localization, the button would be labeled "Start Virtual Camera".
- When you click this button, the current output (either the preview output or the broadcast depending on studio mode) is directed to the Virtual Camera Device. The button text then toggles to the localized "Stop Virtual Camera" label.
- When the user clicks into the settings of their video conferencing app, they will see an entry in the list of cameras labeled "OBS Virtual Camera" (no localisation). When they choose this, the OBS output will be fed from this "device" to the app.
- When the user clicks "Stop Virtual Camera", the virtual device no longer receives frames from OBS. It is up to the consuming application what to do in this state.

# Drawbacks

Platform-specific driver code would require maintenance as operating systems (especially in more recent times) tend to limit access (or require user permission) to access/provide to AV hardware. This especially will mean more testing/debugging will be required each time Windows, macOS, and officially supported Linux flavours (currently Ubuntu) provide a major update.

# Alternatives

* Pursue a more generic output interface and (encourage someone else to?) build a generic virtual camera to consume it. E.g. output via RTSP, NDI and have a "NDI Cam" that has nothing to do with OBS.
