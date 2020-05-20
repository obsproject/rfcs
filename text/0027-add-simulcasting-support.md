# Summary

Add the ability to simultaneously stream to multiple services (i.e. simulcasting support).

# Motivation

The ability to simultaneously stream to multiple services is [currently the most requested feature](https://ideas.obsproject.com/?view=most-wanted) by OBS users \[ see [1](https://ideas.obsproject.com/posts/7/multi-service-streaming-output), [2](https://obsproject.com/forum/threads/simultaneous-broadcasting-to-multiple-streams.76403/), and [3](https://obsproject.com/forum/threads/multi-streaming-to-more-than-one-platform.68698/) \]. However, this feature has not been implemented in OBS for various historical and technical reasons. Since most streamers need to stream to multiple platforms, several work-arounds have been developed over the years to navigate this limitation of OBS – the most popular workaround being the use of paid third party simulcasting services. This increases complexity (and running cost) for users who are interested in using OBS for simulcasting, especially for those users who do not need the additional features provided by third-party simulcasting platforms.

This RFC proposes a way to add simulcasting support to OBS.

# Detailed Changes

## Overview of changes
Several changes need to be made to support this new feature, including UI, frontend apis, backend and configuration. The high-level idea of how we plan to implement this feature is as follows:

1. Add the ability to create multiple stream output settings, each with a unique name.
2. In the stream settings page, a user can then create multiple stream settings and select what output setting to use for each stream based on its unique name.
3. We also plan to expose the simulcasting feature in the current OBS APIs as needed.

## UX: Changes to the output settings window
* Add a list showing the created outputs.
* Add buttons to create a new output and remove a selected output.
* When an item in the list is selected, the output settings for the selected output are displayed.
* A similar change will be applied to both simple and advanced output windows.
![new output settings window](https://user-images.githubusercontent.com/4733470/82475633-c93bc500-9a9a-11ea-9571-b606f2fe131c.png)

##  UX: Changes to streaming settings window
* Similar to the output settings window changes, add a list showing the created streams.
* Also add buttons to create a new stream setting and remove a selected setting.
* Add a dropdown to the stream setting for selecting what output setting to use for streaming.
* The changes above will also be applied to the auto configs page.
![streaming settings window](https://user-images.githubusercontent.com/4733470/82476027-6b5bad00-9a9b-11ea-964a-3d2f68ae65c0.png)

## UX: Changes to the status bar
* Show dropped frames rate for all streams instead of a single stream as done today.
* When we hover over the displayed number (i.e. the colored region in the picture below), a tooltip appears showing the stream name. 
![status bar](https://user-images.githubusercontent.com/4733470/82476359-e9b84f00-9a9b-11ea-9c6f-d572abf5d18f.png)

## Config: Changes to service.json configuration file
The [service.json](https://github.com/obsproject/obs-studio/blob/1bfe4614734e961d34e46f73fd20d7e22855a4be/UI/window-basic-auto-config.cpp#L31) file currently only stores configuration (as json) for a single streaming service. An example of the current format is shown below. Two fields are added to the config –– a non-mutable id assigned to the each service definition at creation time, and a mutable name which is meant to be displayed on the UI list widget described earlier.

```json

{
    "settings": {
        "bwtest": false,
        "key": "XXXX-XXXX-XXXX-XXXX",
        "server": "rtmp://example.com/",
        "use_auth": false
    },
    "type": "rtmp_custom"
}

```

To support multiple streams, we instead store an array of output services as shown below.

```json
{
 "services": [
                 {
                     "settings": {
                         "bwtest": false,
                         "key": "XXXX-XXXX-XXXX-XXXX",
                         "server": "rtmps://example.a.com/",
                         "use_auth": true
                     },
                     "type": "rtmp_custom",
                     "name": "Example A Streaming Service",
                     "Id": 2
                 },
                 {
                     "settings": {
                         "bwtest": false,
                         "key": "XXXX-XXXX-XXXX-XXXX",
                         "server": "rtmp://example.b.com/",
                         "use_auth": false
                     },
                     "type": "rtmp_custom",
                     "name": "Example B Streaming Service",
                    "id": 1
                 }
             ]
}
```

To avoid breaking currently saved settings when loading, we determine if the current setting is of the first or second kind and then load appropriately.

## Config: Supporting multiple outputs
Add name and id fields to each output similar to the service.json

## Backend: Internal changes to support multiple streams
* Change the [current stream output](https://github.com/obsproject/obs-studio/blob/7993179466cffcccd7973c2105563d9a6924f75d/UI/window-basic-main-outputs.hpp#L9) from a single to a list of stream outputs – one for each stream.
* Each stream output will be responsible for each individual stream.
* Change [StartStreaming(obs_service_t *service)](https://github.com/obsproject/obs-studio/blob/7993179466cffcccd7973c2105563d9a6924f75d/UI/window-basic-main-outputs.cpp#L683) to StartStreaming(std::vector<obs_service_t *> services)
* When [StartStreaming(services)](https://github.com/obsproject/obs-studio/blob/7993179466cffcccd7973c2105563d9a6924f75d/UI/window-basic-main-outputs.cpp#L683) is called, a new stream output is created for each service and added to the list of stream outputs.
* StartStream is only considered successful if all streams are successfully started.
* [StopStream(bool force)](https://github.com/obsproject/obs-studio/blob/7993179466cffcccd7973c2105563d9a6924f75d/UI/window-basic-main-outputs.cpp#L1014) stops all output streams

## API: Changes to frontend & backend APIs
* Define an **obs_frontend_service_list** struct and **obs_frontend_service_list_free** function similar to [obs_frontend_source_list](https://obsproject.com/docs/reference-frontend-api.html#c.obs_frontend_source_list) which will simply be a dynamic array of services (and function to free the list).

* Instead of having the **obs_frontend_set_streaming_service** as we currently do, we propose adding two new APIs **obs_frontend_add_streaming_service** and **obs_frontend_remove_streaming_service** and which will both accept an obs_service_t pointer.

* Rename **obs_frontend_get_streaming_service** to **obs_frontend_get_streaming_services** and it will now accept a pointer to **obs_frontend_service_list**. <br />
  
  Current definition:
    ```c
      obs_service_t *obs_frontend_get_streaming_service(void)
    ``` 
  Returns a new reference to the current streaming service object.
  
  New definition:
    ```c
      void obs_frontend_get_streaming_services(struct obs_frontend_service_list *services)
    ```
  The services param is a pointer to an **obs_frontend_service_list** structure to receive the list of reference-incremented services. Release with **obs_frontend_service_list_free()**.

* Rename **obs_frontend_save_streaming_service** to and **obs_frontend_save_streaming_services** <br />

  Current definition:
  ```c
    void obs_frontend_save_streaming_service(void)
  ```

  Saves the current streaming service data. 

  New definition:
  ```c
    void obs_frontend_save_streaming_services(void)
  ```
  Saves the current streaming service data.

* We can leave the current APIs in place and mark them as OBS_DEPRECATED.


# Drawbacks

Streaming to multiple services will use up more compute resources than streaming to a single service.

# Additional Information

Note that these changes only cover simulcasting, and it allows for a single recording. Extending OBS to perform multiple recordings, though feasible, is out of scope for this RFC.