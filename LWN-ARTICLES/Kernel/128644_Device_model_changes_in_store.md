---
title: Device model changes in store
url: https://lwn.net/Articles/128644/
date: "March 22, 2005"
category: Device model; klist
author: 
---

The Linux device model is a core subsystem which implements various useful device-level functions, including reference counting, sysfs, hotplug event generation, and more. Some of the lower-level device model subsystems were covered in the [LWN driver porting series](<https://lwn.net/Articles/driver-porting/>); there is also a device model chapter in [LDD3](<https://lwn.net/Kernel/LDD3/>). All of that nice documentation is now threatened with obsolescence, however; a number of device model changes are currently in the works. 

### Class code changes

The device model ["class" code](<https://lwn.net/Articles/31370/>) is the mechanism behind `/sys/class`. Its purpose is to make information about devices (and more) available in a way which is independent of the underlying hardware topology. The largest use of classes, probably, is to export device numbers which can be used (by tools like `udev`), to create device nodes when hardware is added to the system. The class subsystem, like much of the device model code, has proved to be somewhat complex and error-prone to work with. 

As a way of making things easier, the "`class_simple`" interface was added some time ago. This interface handles much of the boilerplate code for allocation of `class` structures, management of attributes, and life cycle management. Greg Kroah-Hartman has now [concluded](<https://lwn.net/Articles/127860/>) that `class_simple` was the sort of interface which was needed from the outset, so he has posted a set of patches which move the full `class` interface in that direction. 

With the new interface, `class` structures are no longer created by the driver. Instead, one is allocated with a call to: 

```
struct class *class_create(struct module *owner, char *name);
```

This function will allocate the structure, initialize it, and register it with the given `name`. When the structure is no longer needed, it can be handed to `class_destroy()`, which will unregister it, decrement its reference count, and, eventually, get rid of it. 

The `class_device` structure, which represents a single device under a class, also gets a dynamic allocation function: 

```
struct class_device *class_device_create(struct class *cls, dev_t devno,
                                                 struct device *device, 
                                                 char *fmt, ...);
```

The `devno` parameter is the device number; it is used to create the `dev` attribute for the class device entry. If `device` is non-`NULL`, it will be used to create a symbolic link to the appropriate entry under `/sys/devices`. The name of the device is passed in as a `printk()`-style format string. 

Interestingly, a `class_device` structure is not destroyed directly; instead, the driver should call: 

```
void class_device_destroy(struct class *cls, dev_t devno);
```

The class code will find the `class_device` entry corresponding to the given device number and get rid of it. 

The new functions may just look like some added convenience utilities, but Greg's long-term intent is to phase out the current class interface in favor of the new functions. The older versions, he says, are simply too hard to use correctly. Others may agree with this point, but there have been a few objections to this change. It really does represent a different way of doing things with the driver model. 

Under the old scheme, `class` and `class_device` structures are typically embedded within larger, bus-specific (or driver-specific) structures. The reference counting implemented for the class-subsystem structures also worked for the containing structure. Thus the higher-level code, if written right, did not have to implement separate reference counting and life cycle management for its own structures. 

The new way of doing things makes it impossible to embed the class structures in this way; they must, instead, be allocated separately and accessed via a pointer. So the bus-level or driver-level code must do its own reference counting for its own structures. The changes are often small. The patch to change the USB subsystem over, for example, adds a `kref` to `struct usb_bus`. Then, the function for obtaining a reference to a USB bus structure is changed this way: 

```
struct usb_bus *usb_bus_get(struct usb_bus *bus)
       {
     	if (bus)
       -		class_device_get(&bus->class_dev);
       +		kref_get(&bus->kref);
     	return bus; 
        }
```

So the changes are not all that huge, but, if all users of the old interface are to be switched over, new reference counts will have to be added in a number of places. If this change goes through, look for similar changes to other parts of the device model API in the future. 

### Delaying hotplug events

When a device is added to (or removed from) the system (more specifically, when a `kobject` representing that device is added or removed), the kernel generates a hotplug event to inform user space. That event is passed on to a tool like `udev`, which looks up the device number in sysfs and creates the appropriate device node(s). As it turns out, however, the hotplug event is generated before the sysfs attribute containing the device number is created. So, if the timing works out badly, `udev` must spin in a loop waiting for the attribute it needs to show up. 

Kay Sievers has posted [a series of patches](<https://lwn.net/Articles/128230/>) which addresses this problem by making a change to the `kobject` API. In particular, `kobject_add()` and `kobject_del()` no longer generate hotplug events. Kernel code which uses those interfaces must explicitly generate hotplug events itself through calls to `kobject_hotplug()`. This change would appear to put extra work on higher-level code, but it has an important advantage: the `kobject_hotplug()` call can be made _after_ the relevant sysfs attributes have been set up properly. Making the system as a whole work more smoothly is worth a small amount of added complexity. 

The wrapper functions `kobject_register()` and `kobject_unregister()` have not been changed, and still generate hotplug events. 

### Locking and klists

The device model implements a shockingly complex data structure which must be protected against concurrent access. Much of that protection is handled by a reader-writer semaphore (rwsem) kept in the top-level `subsystem` structure. There has been a slow stream of patches aimed at removing that rwsem for a while now; it is seen as inelegant and a performance bottleneck. Pat Mochel has just posted [a series of patches](<https://lwn.net/Articles/128645/>) aimed at pushing this process forward some more. 

Many of the structures needing for locking are linked lists. In the current device model code, the standard kernel list type is used to implement these lists. Pat has decided that a new list type, which he calls a `[klist](<https://lwn.net/Articles/128646/>)`, is the right way to deal with many of the locking issues in the device core. The `klist` is built on the regular `list_head` type, but it adds some interesting properties. 

The first of those properties is that the real head of the list has a different type (`struct klist`) from the entries in the list (`struct klist_node`). So klists are not explicitly circular lists; they have a clear starting point. The `klist` structure contains a spinlock which is used to serialize access to the list itself (but not to the individual nodes on the list). 

The set of basic klist functions is rather smaller than the equivalent `list_head` functions: 

```
void klist_init(struct klist *list);
        void klist_add_tail(struct klist *list, struct klist_node *node);
        void klist_add_head(struct klist *list, struct klist_node *node);
```

The `node` structure is initialized automatically when it is added to the list, so there is no need for the caller to worry about it. 

The `klist_node` structures contain their own reference count; as long as the count is non-zero, the node will continue to be part of the list. There are two removal functions: 

```
void klist_del(struct klist_node *node);
        void klist_remove(struct klist_node *node);
```

A call to `klist_del()` will decrement the `node`'s reference count and return immediately; the entry may still exist on the list at that point. `klist_remove()` is like `klist_del()`, but it will, if necessary, sleep until the last reference has been given up and the `node` has actually been taken off the list. 

Working through a klist requires the creation of an iterator structure - `struct klist_iter`. Iteration is started with a call to one of: 

```
void klist_iter_init(struct klist *list, struct klist_iter *iter);
        void klist_iter_init_node(struct klist *list, struct klist_iter *iter,
                                  struct klist_node *node);
```

The first form starts iteration at the beginning of the list, while the second can be used to start at an arbitrary entry within the list. Stepping through the list is accomplished with: 

```
struct klist_node *klist_next(struct klist_iter *iter);
```

This function will return a pointer to the next node in the list, if there is one. It also will grab a reference to that node, so that it will not go away while the iterating code is working with it. Among other things, that feature makes it safe to call `klist_del()` on a node while iterating through the list; that node will continue to exist (at least) until `klist_next()` is called. Also implied is that calling `klist_remove()` while iterating through a list is a very bad idea; it will wait rather longer than the caller intended. 

Iteration is ended with: 

```
void klist_iter_exit(struct klist_iter *iter);
```

This function will release the reference on the last node returned from `klist_next()` (if any) and stop the iteration. 

The klist code drew [an objection](<https://lwn.net/Articles/128674/>) about the obfuscation caused by all of the device model "kfoo stuff." Pat [responds](<https://lwn.net/Articles/128675/>) that the klist code is, instead, a step toward cleaning up some of that obfuscation. There were not a whole lot of other comments on this patch series. 

It's worth noting that, as of this writing, none of the patches described above have been merged. They are sufficiently disruptive that, at this point, they may have to wait until 2.6.13.
