# Concurrent Coding



## When versioning  is not enough

Using a **SVN** or **GIT** as version control system is very useful, but when several developers are working within a same Sprint on the same bunch of source files, it can rapidly become messy and lead to endless resolutions of merge-conflicts.

## Thinking in terms of controllers rather than objects


Instead of splitting development into functions or methods, think of code interaction by allowing controllers to call each other can be very helpful

Instead of calling a method, a script / controller can request the result from (or invoke action to) another controller.

As controllers are described within distinct files, this practice decreases the risk of conflict when code is updated.

## Implementation

This practice applies to most frameworks, both front-end and back-end:

* Angular : by defining components and related controllers
* Zend / Symfony / Laravel: controllers
* Qinoa : native

