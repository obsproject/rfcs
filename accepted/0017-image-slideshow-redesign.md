# Summary

Re-design the image slideshow to remove its current limitations.


# Motivation

In its current implementation, the image slideshow can only play a low number of images.
The user is not made aware of that fact, they just notice that only the first images are shown in the
slideshow.

It should be able to show all of the selected images.

A minor issue that may be left out of this RFC is the way random playback is behaving. Currently it picks the 
next image completely random. This may result in some images getting shown multiple times while others may get 
shown seldom or not at all. The current implementation also suffers from modulo bias which means the depending 
on the number of images, the first few images may be "favored" by the algorithm and shown more often. The 
author's expectation for random playback is to play back the whole list once, but in random order (shuffled),
thus each image is shown exactly once before the first repeat is seen.


# Detailed design

## Main design 

* The slideshow gets a list of files the user has configured to play.
* At the start, the list is copied to a new list, the playback queue.
  * The reason for this is to support an improvement in the way random playback is handled (this is discussed below).
  * It also allows to remove invalid/missing images from the queue but keeps the user's intention (play this 
    file), so at the next opportunity the slideshow may try to handle this file again.
* There is a cache/preload buffer that stores only the decoded images that are needed in the near-future or that
  have been used recently (to handle "go back an image").
* The video tick function fetches images from the cache.
  * If at a tick an images had to be shown but it wasn't available in the cache yet, the transition is deferred 
    and tried again at the next tick. This should happen seldomly, for example right at the start when the cache
    was not yet able to load the current image.
* A dedicated thread is responsible for managing the cache/preload content: images are loaded as required 
  according to the current position in the queue and entries that are no longer needed are evicted, freeing
  their resources.
  * This thread is waiting for an event so that it's sleeping when there is no work to do and woken up once the
    index into the playback queue is advanced.
  * First, it ensures that the image for the current playback position is loaded.
  * Then, it ensures that the next few images (subject to compile-time constant or setting) are loaded.
  * Then, it ensures that the previous few images are loaded to support going backwards.
  * Lastly, it evicts all other images from the cache, thus unloading their resources if there are no more 
    references (for example, from a transition).
* Increasing/decreasing the index into the queue signals the event that wakes up the caching/preloading thread
  to do its work.


## Random playback

* At the start of playback, the file queue is shuffled.
* The index then advances in the queue as it would if no randomization was requested.
* If the end of the queue is reached, it is re-shuffled.
* To avoid having images shown immediately after another when the queue is shuffled again, we may ensure that
  the first few items of the newly shuffled queue are not part of the tail of the previous shuffled queue.
  * For example, ensure that the images at the start of the newly shuffled queue are not part of the last 20% or
    so of the previous queue.


# User UX

No changes are needed, unless it's decided to support both the current (fully random) and new (shuffled queue)
random playback concepts.


# Drawbacks

It's possible that an image is not yet ready when the elapsed time demands a transition to the next image. 
Displaying that image is then deferred until the image has been loaded.



# Unresolved questions

Both the current and hereby proposed implementations make OBS stall after configuration changes and on app-load.
This is because all images in the list are loaded immediately: the current implementation does so because it
only preloads and cannot load on-demand, the proposed implementation because the maximum width and height of the
images need to be determined. Due to the memory limitations, the current implementation stops after a few images
thus limiting the stall; the hereby proposed implementation does not stop and thus may stall OBS for a longer
time period.

It's unclear to the author how to resolve this yet:

* Can we get rid of determining the image sizes altogether?
* If not, is it possible to offload this work into a thread and update the slideshow source once the values have
  changed?

An additional cache may help improve the situation if we are not able to get rid of looking at each image: this
cache would store the file paths, last file modification date and the image size. When the file list needs to
be evaluated again after settings have changed, this cache may be consulted and the corresponding files may be
`stat`ed to determine whether the cache entry is still valid. If there is none or the file has changed, only
then would the image need to be loaded to determine its size.


# Additional Information

* [Initial Mantis issue #1621](https://obsproject.com/mantis/view.php?id=1621)
* [Cancelled pull request #2694](https://github.com/obsproject/obs-studio/pull/2694)
