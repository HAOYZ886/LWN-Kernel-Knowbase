---
title: Some more 2.6.12 API changes
url: https://lwn.net/Articles/127858/
date: "March 16, 2005"
category: Device model; Sorting
author: 
---

A few more changes to the 2.6 internal kernel API have been merged since [last week's summary](<https://lwn.net/Articles/126823/>). 

The driver model API has seen a couple of small changes. `kref_put()` no longer returns `void`: 

```
int kref_put(struct kref *kref, void (*release)(struct kref *kref));
```

The (new) return value is normally zero, but will be nonzero if the kref was actually removed. Note that a zero return does not imply that the kref is still valid; somebody else may have done the last `kref_put()` call in the mean time. 

The `kset` type now has its own internal spinlock. That means that a kset is no longer required to be part of a subsystem. 

Greg Kroah-Hartman has proposed [a rather wider set of changes](<https://lwn.net/Articles/127860/>) to the driver model class code. Essentially, he is pushing all users over to a form of the "class_simple" interface, and getting away from the original class implementation, which was hard to use correctly. These changes have not yet been merged, however. 

The kernel has long held a variety of special-purpose sorting functions. These have now been replaced by a generic heap sort utility written by Matt Mackall. It's interface is: 

```
void sort(void *base, size_t num, size_t size, 
                  int (*compare)(const void *a, const void *b),
                  void (*swap)(void *a, void *b, int size));
```

Here, `base` is the array of items to sort; it contains `num` items of `size` bytes. The `compare()` function returns the integer equivalent of `a-b`; `sort()` will sort the array in ascending order as dictated by `compare()`. The `swap()` function is optional; it can be provided if the caller knows a faster way to exchange two elements in the array.
