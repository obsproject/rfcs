- Start Date: 2020-01-03
- RFC PR: #5

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-refresh-toc -->
**Table of Contents**

- [Summary](#summary)
- [Motivation](#motivation)
- [Design](#design)
    - [UX](#ux)
    - [Functionality](#functionality)
        - [Core Functionality](#core-functionality)
        - [Memory](#memory)
        - [v1](#v1)
        - [v2](#v2)
        - [Specification](#specification)
- [Requirements](#requirements)

<!-- markdown-toc end -->


# Summary
Create a undo system that is capable of recording the state of OBS actions to
easily revert the state forwards or backwards in time.

# Motivation
When working with large and intricate processes - whether it be scenes, sources,
etc. - accidents are bound to happen. Currently, there is no way to undo
committed actions. As a result, work can easily be lost at the expense of hours
of effort.

Implementing a proper undo/redo system would be a drastic increase in
quality of life for the majority of users.

# Design
## UX
The UX would model that of most other applications. Under edit, there would be 2
options: 'Undo' and 'Redo'. Appended to these actions, is a descriptor of what 
action is being (re/un)done alongside on what object. When an action is committed,
for instance deleting a scene, the undo button becomes activated, allowing the
user to select it.


When activated, it undos the action and then activates the Redo button, which
allows the user to essentially 'undo' their previous undo. If the user makes
multiple undos, then the users would also have multiple redos available.

When a user undos and then commits another action, the redos are no longer
possible as the tree has diverged into a new path.

Also, like many other applications, the undo/redo keys should be binded to Ctrl-Z
and Ctrl-Y

## Functionality
### Core Functionality
The undo/redo system work by saving states. However, saving the entire state of
OBS would use up more memory than necessary when a small action is committed. In
order to prevent this, only the state of the object the action was committed on
would be saved, alongside the action committed. This will allow more seamless
integration into OBS without disturbing the flow, and also keep the amount of
memory low. 

Note, the same methodology would work for the redo system.

The method to create this list would be through the use of two dequeues. This
allows seamless popping and inserting from the front of the list. This would
essentially act lack a stack.

The object that would go into the dequeues is an undo_redo\_t which contains a
callback function to be called when undo/redo is activated. Several functions
have cleanup that is needed to be done, so a third callback function can be
specified that will call when the object is either cleared from the redo, or
the undo\_stack goes out of scope and gets cleaned up.

### Memory
The current design of only saving the state of the local area is intended to
reduce memory consumption. However, over the course of a long lifetime, lots of
memory would eventually be used by having a long tree. After a certain length n 
(perhaps customizable?) the end of the chain should be saved to disk and removed
on close.

Having disk read/write operations for each action would be costly. In
order to reduce that load it should only save/read in sets of n when
it needs to.

### v1
Note: Items checked off are implemented in POC.
The core scope should include:
- [X] Scenes
  - [X] Add
  - [X] Delete
  - [X] Rename
  - [X] Duplication
  - Transitions
    - Add
    - Remove
    - Properties
- [X] Sources
  - [X] Add
  - [X] Delete
  - [X] Rename
  - [X] Transforming
- [X] Scene Collections
  - [X] Add
  - [X] Removal
  - [X] Switching
  - [X] Rename
- Audio
 - Volume Settles (i.e stops after time)
- [X] Filters
    - [X] Add
    - [X] Remove
    - [X] Rename
    - [X] Properties

### v2
- Order Changes
- Grouping
- Property Changes
  - Changes occur in local undo stack, then gets simplified to single undo in main.
- Other Small changes that could positively impact UX

### Specification
- Properties
 - On Save/Cancel
- Transforms
 - Mouse Release
 - On Close
- Grouping
 - Insert
 - Removal

# Requirements
This section is a clear guideline on what successful completion is. This feature
can be split up into two segments, necessities, and benefits. The necessities 
implement undo/redo for core features where users spend their time making adjustments,
such as scenes and sources. Benefits are undo/redo for items that are not strictly necessary,
but would be nice on the user side, such as undoing drag and drop order. Successful completion must include all 
items outlined in the scope, **at a minimum**. 

On top of having the core features implemented, they must be thoroughly tested to
ensure consistent validity and cohesion. It should be tested enough
to ensure that all data retains its integrity and can not corrupt data. It must
also be memory safe and not cause any memory leaks. It should also be minimally invasive to the rest
of OBS, as to not cause any other UB or bugs in the application as a whole.
