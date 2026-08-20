---
title: A big set of driver core changes
url: https://lwn.net/Articles/140787/
date: "June 21, 2005"
category: Device model; Sysfs
author: 
---

Greg Kroah-Hartman has gotten 2.6.13 off to a good start with [a massive set of driver core patches](<http://lwn.net/Articles/140764/>). There are a fair number of API changes that come with this patch set, so the whole thing is worth a look. In-tree code has been fixed to use the new API, but, as always, maintainers of external code are on their own. 

Two of the more significant changes were [covered here last March](<https://lwn.net/Articles/128644/>). The interfaces have not changed since then, so that coverage will not be duplicated. The first of these changes is the complete rework of the "class" API. The interface known as "class_simple" turned out to be the best way to work with classes, so Greg reworked it as _the_ class API, changing everything as he went. The interface known as class_simple is no more, but the new class API looks much like class_simple used to. The other change is the addition of the "klist" type: an extension to the kernel linked list type which includes its own, built-in reference counting and locking. 

The next change is in the prototypes of the `store()` and `show()` callbacks for device attributes. These callbacks now look like: 

```
ssize_t (*show)(struct device *dev, struct device_attribute *attr,
                        char *buf);
        ssize_t (*store)(struct device *dev, struct device_attribute *attr,
                         const char *buf, size_t count);
```

In each case, the callbacks have picked up a pointer to the actual attribute being accessed, allowing one callback to handle multiple attributes. 

There have been a number of internal changes to device model data structures which really shouldn't affect other code, but which might anyway. Various internal lists have been removed; in some cases, they have been replaced with klists. And a number of character pointers are now explicitly `const` pointers. 

Code wanting to look through the devices bound to a driver can use a new function to iterate through the list: 

```
int driver_for_each_device(struct device_driver *driver, 
                                   struct device *start, 
    			       void *data, 
                                   int (*fn)(struct device *, void *));
```

This function will call `fn()` for each device bound to the given `driver`, stopping at the end of the list or when `fn()` returns a non-zero value. 

Inodes in sysfs now have an `i_op->setattr()` function, meaning that their permissions can be changed and those changes will last for as long as the system runs. Changing of sysfs permissions was never really supported in the past; it would work for a bit, but the permissions could be reverted at seemingly random times. This is not really an API change, but creators of sysfs attributes should bear in mind that the permissions on those attributes might be changed from their original values.
