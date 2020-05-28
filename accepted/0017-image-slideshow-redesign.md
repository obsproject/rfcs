# Summary

Re-design the image slideshow to remove its current limitations.


# Motivation

In its current implementation, the image slideshow can only play a low number of images.
The user is not made aware of that fact, they just notice that only the first images are shown in the
slideshow.

It should be able to show all of the selected images.


# Detailed design

## Main design 

* The slideshow gets a list of files the user has configured to play.
* There are two load modes, which the user can choose from: `preload` and `on-demand`.
  * The `preload` mode behaves like the current slideshow does: images are loaded at start until a (currently
    hard-coded) memory limit is reached. Remaining paths are discarded.
  * In `on-demand` mode, a thread is spawned that loads a small amount of images into a cache. As the slideshow
    progresses, images are evicted from the cache and soon-to-be-required images are loaded on this thread.
* On update, the list of files to play is read.
* A central list, `entries` is derived from the list of files.
  * An entry stores the path and loaded image source. In `on-demand` mode, the source may be NULL. In `preload` 
    mode, the source is always set.
  * Access to the list (and most other properties) is only done with a mutex.
    * Even in `preload` mode, there are at least two threads that need to access the cache: the video tick
      thread and the thread running `update`.
    * The locked code parts should be kept as short as possible. No expensive operations should be done while
      the mutex is held.
* Video tick:
  * Transitions, stop, pause, etc. are handled in the same way as the current slideshow.
  * If the elapsed time reaches the transition to the next entry:
    * The next cache entry is read and displayed.
    * In `preload` mode, the entry will always exist.
    * In `on-demand` mode, it's possible that the worker thread was not able to fetch the next image in time.
      In this case, it is remembered that a transition was supposed to happen and is retried in the next video 
      tick. The elapsed time is treated as if the transition was successful.
* Config update:
  * To make it easier to handle both modes and also to handle switching modes correctly, all entries are 
    unloaded on config change.
  * The `file_paths` list is read/built.
  * The `entries` list is created by iterating over the `file_paths`.
    * In `preload` mode, the image sources are immediately loaded until the hard-coded memory limit or the end
      of the `file_paths` is reached, whichever comes first.
    * In `on-demand` mode, entries for all file paths are created but no image source is loaded yet.
  * The current item is set to the first entry.

## Preload mode

* Update:
  * As explained in the _Main Design_ section, the `entries` list is built by iterating over the `file_paths`
    and loading each file.
  * Entries for which the image cannot get loaded are discarded.
  * Once the memory limit is reached, loading stops. Thus, `entries` may be shorter than `file_paths`.
  * The `entries` list thus stores all loaded images and does not evict them at runtime in this mode.
* Current item updates:
  * Changing the current item (advancing to the next, previous or a random item) does not need any special
    handling. The index can be updated directly.

## On-demand mode

* Update:
  * As explained in the _Main Design_ section, the `entries` list is built by iterating over the `file_paths`
    but no images are loaded at that point.
  * The cache thread is signaled to start loading images.
* Current item updates:
  * In addition to the cache, there is a cache index list called `cache_queue`. It tracks which items in the
    cache should be populated.
  * To track the actually populated entries and calculate what to do (evict, load), there is a cache index
    list called `cached_entries_set`.
      * While the `cache_queue` may contain duplicate indices and are in any order, the `cached_entries_set` is
        an ordered, deduplicated list.
  * Advancing to the next item modifies the cache index list:
    * The cache index list `cache_queue` stores the last few indices that were displayed (to go back), the
      current index and a few upcoming indices. It thus controls the number of items to load in the cache.
    * For example, if two items for history and three items for prebuffering are chosen, the cache list has a 
      size of 2 + 1 + 3 = 6.
    * Duplicate indices are allowed in `cache_queue` (after all, a slideshow with just one image must be 
      handled as well).
    * Every time the "next" image is requested, the cache index list is first shifted by one, discarding the 
      oldest index.
      * For random playback, a new random index is placed at the "end" of the cache index list.
      * For sequential playback, the next index in the sequence is placed at the "end" of the cache index 
        list.
* The worker thread operates as follows:
  * The thread runs in an endless loop until its thread gets cancelled.
  * It waits on an event to get triggered so it can sleep when there's nothing to do.
  * It inspects `cache_queue`, derives a sorted, deduplicated list of indices and calculates which entries
    to evict or load by comparing with `cached_entries_set`.
  * All entries that can be evicted have their sources released immediately.
  * However, only one image is loaded per iteration.
    * The next image to load should consider the current item and always try to load that first, if required.
    * Only one image is loaded per iteration since it may take some time and the current image index may have 
      been changed.
    * Once the image is loaded, its source is placed into its corresponding entry.

# User UX

The following additions are needed to the current slideshow UI:

* The "Bounding Size/Aspect Ratio" cannot support "Automatic" any more. The user has to chose a resolution.
* A selection for the load mode `preload` and `on-demand` mode.

# Unresolved questions

For `preload` mode, the user should be alerted somehow if not all images in the list could get loaded due to
the memory limit.


# Additional Information

* [Initial Mantis issue #1621](https://obsproject.com/mantis/view.php?id=1621)
* [Cancelled pull request #2694](https://github.com/obsproject/obs-studio/pull/2694)
