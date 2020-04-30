- Start Date: 2020-03-25
- RFC PR: #15
- Related Github Issue: https://github.com/obsproject/obs-studio/issues/25685

# Summary

Add the ability to render output to a virtual camera device on Linux, Mac, and Windows.

# Motivation

OBS is a powerful set of tools to manipulate live video streams that natively supports output to popular streaming services as well as rendering to a local video file. There are a huge number of people who engage in 1:1 or small-scale streaming using video conferencing software like Zoom or Google Hangouts. Many of these people have similar needs to those of traditional streamers, but for one reason or another cannot switch the video conferencing software they use (social graph, corporate policy).

OBS can meet this need by creating a virtual camera device and outputting to that device such that programs capable of consuming camera input can now consume OBS input with no extra development.

Similar functionality: 
* OBS can be extended by the [OBS-VirtualCam plugin](https://obsproject.com/forum/resources/obs-virtualcam.539/) (Windows) and [obs-v4l2sink plugin](https://github.com/CatxFish/obs-v4l2sink) (Linux). Currently, there is no OBS solution for macOS.
* [Wirecast includes virtual camera support](http://www.telestream.net/pdfs/user-guides/Wirecast-8-User-Guide-Windows.pdf) on both Windows and Mac.
* [Webcamoid](https://webcamoid.github.io/) is an existing open-source cross-platform virtual camera application, which may have some useful tidbits when investigating implementation details for each platform.

# Detailed design

Feature Set:

* A button is added to the Output Controls dock labeled "Start Virtual Camera", which will toggle the start/stop status of the virtual camera output. While virtual camera output is active, the label should read 'Stop Virtual Camera".
* The virtual camera appears in the system listed as "OBS Virtual Camera".
* The virtual camera outputs at the same resolution and framerate as the OBS rendered output.
* A section is added to OBS settings providing the following settings:
    * A checkbox to indicate if the user wishes to start the virtual camera output automatically when OBS starts
    * A checkbox to flip the virtual output horizontally
* In the Output settings tab, the warning that appears when outputs are active is augmented to list all active outputs so that users know which outputs they need to disable in order to edit output settings.
* When the virtual camera output is not active, the virtual camera device shows a static image to consumers indicating to the consumer that output is not started. This image meets four requirements:
    * It allows the consumer to smoothly switch from non-active to active content without having to worry about what order the applications are started in.
    * It informs the user that the output is not started yet, and needs to be started before output will be sent from OBS.
    * It is aesthetically pleasing such that, if the output is accidentally sent to viewers (for example, if the user fails to start the virtual output before joining a Zoom call), it is not unnecessarily gaudy or technical.
    * It requires minimal localization, or provides a way to localize any displayed text on the image

## Platform specific implementations

* On **Windows**, the implementation of this plugin should leverage [libdshowcapture](https://github.com/obsproject/libdshowcapture). OBS already depends on libdshowcapture, so using this existing library service to limit the number of extra dependencies that OBS needs
* On **macOS**, OBS will need a plugin to output over IPC to a CoreMediaIO DAL plugin that is registered on the system upon OBS install. This will require repackaging the installer as a `.pkg` instead of a compressed `.app`.
* On **Linux**, the most straightforward way to implement this functionality would be to output via [v4l2sink](https://gstreamer.freedesktop.org/documentation/video4linux2/v4l2sink.html?gi-language=c). Note that this still requires the user to have [v4l2loopback](https://github.com/umlaeute/v4l2loopback) installed to consume this output. For package installations, a dependency should be placed on v4l2loopback such that it is installed by the package manager when OBS is installed.

## Technical considerations

- This RFC is explicitly for an OBS-specific implementation of output from OBS to a virtual camera. A generic middleware solution to this problem is out of scope for this problem.
- The virtual camera will need to be registered with the system upon OBS installation.
- The virtual camera should be accessible by third party applications that support webcams/capture cards
- OBS supports running multiple instances of itself. If two instances of OBS run simultaneously, the first instance to output to the device should "win". The second instance should give an error saying that the output device is already in use by another instance of OBS.

## User UX

* When installing, OBS should OBS should install the platform specific driver to enable virtual camera output on the system
* Initial configuration should require very little input from the user, if any. Resolution and framerate of the camera will be pre-defined by the Video settings of OBS, and no effects outside those already provided by OBS will be specially exposed for this output.
* Add a button in the OBS UI below "Start Recording" and above "Studio Mode". In English localization, the button would be labeled "Start Virtual Camera".
* When you click this button, the current output is directed to the virtual camera device. The button text then toggles to the localized "Stop Virtual Camera" label.
* When the user clicks into the settings of their video conferencing app, they will see an entry in the list of cameras labeled "OBS Virtual Camera" (no localization). When they choose this, the OBS output will be fed from this "device" to the app.
* When the user clicks "Stop Virtual Camera", the virtual device no longer receives frames from OBS, and instead receives a static image defined by OBS indicating that the output is not active.

# Drawbacks

* The most obvious omission from this RFC is the lack of virtual audio output in addition to virtual camera output. However, though the features are logically closely related, they are unfortunately vastly different technically speaking. As such, we will leave the implementation of virtual audio output to [a separate RFC](https://github.com/obsproject/rfcs/pull/16).
* Platform-specific driver code would require maintenance as operating systems (especially in more recent times) tend to limit access (or require user permission) to access/provide to AV hardware. This especially will mean more testing/debugging will be required each time Windows, macOS, and officially supported Linux flavours (currently Ubuntu) provide a major update.

# Alternatives

* Pursue a more generic output interface and (encourage someone else to?) build a generic virtual camera to consume it. E.g. output via RTSP, NDI and have a "NDI Cam" that has nothing to do with OBS.
* Adapt Webcamoid's source code for our use. This may end up being more involved than we'd like, however, and might end up making the feature more complicated than we want.
