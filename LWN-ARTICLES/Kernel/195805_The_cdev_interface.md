---
title: The cdev interface
url: https://lwn.net/Articles/195805/
date: "August 16, 2006"
category: Char devices; register chrdev region
author: 
---

Since time immemorial, the basic registration interface for char devices in the kernel has been: 

```
int register_chrdev(unsigned int major, const char *name,
                            const struct file_operations *fops);
        int unregister_chrdev(unsigned int major, const char *name);
```

In the old days, `register_chrdev()` would allocate all 256 minor numbers associated with the given `major`, associating the given `name` and file operations with all of them. If the major number is given as zero, one will be allocated on the fly. The corresponding `unregister_chrdev()` call would release all of those minor numbers. This call asked for the name as a safety measure; if the name did not match that provided when the major number was registered, the `unregister_chrdev()` call would fail. 

In the intense period prior to the release of the 2.6.0 kernel, Al Viro set out to find a way to expand the device number range. One of the problems to be solved was the huge set of drivers which "knew" that minor numbers never went any higher than 255. One option would have been to audit every driver in the tree, ensuring that it did the right thing with minor numbers. Time was in short supply, however, and volunteers to do that particular job were in even shorter supply. So Al took a different approach: he created a new interface for the registration of char devices, then reimplemented the old interface as a compatibility layer which would allocate minor numbers 0..255 for a given major. In this way, unconverted code would continue to work as always, with the kernel guaranteeing that it would never see any minor numbers that it would not have seen before. Over time, drivers could be converted to the new interface, which has a number of advantages. 

As it happens, that conversion never really came to be. Since the old interface continued to work, was familiar, and was a little simpler to use, developers stuck with it. Perhaps more importantly, the long-feared device number shortage never happened. Greater use of dynamic numbers, more generic device interfaces, and the hotplug mechanism all came together to make (most) Linux systems fit easily within the older device number space, to the point that the expanded numbers are rarely used. A quick scan on your editor's system reveals exactly three minor numbers greater than 255, all under `/dev/bus/usb`. So there has been no strong reason to convert to the new character device interface. 

Recently, Alexey Dobriyan noticed that `unregister_chrdev()` no longer checks the `name` argument, so he posted [a patch](<http://lwn.net/Articles/195617/>) which removes that argument, fixing all callers in the process. Your editor suggested that, perhaps, this would be a good time to move those callers to the newer interface, rather than reworking the older, compatibility interface. In response, another developer suggested that better documentation for the new interface would be a good thing to have. To that end, here is a quick overview of how char device registration is meant to be done in 2.6. 

The newer interface breaks down char device registration into two distinct steps: allocation of a range of device numbers, and association of specific devices with those numbers. The allocation phase is handled with either of: 

```
int register_chrdev_region(dev_t first, unsigned int count, 
                                   const char *name);
        int alloc_chrdev_region(dev_t *first, unsigned int firstminor,
                                unsigned int count, char *name);
```

The first form will allocate `count` minor numbers, starting with the major/minor pair found in `first`, and remembering `name` with all of them. The second form is intended for use when the desired major number is not known ahead of time; it will allocate a major number, then allocate `count` minor numbers, starting at `firstminor`. The beginning of the allocated number range will be returned in `first`. The return value will be zero on success or a negative error code on failure. 

A few things are worth noting here. With either version, the major number used could be shared with other, completely unrelated devices. Only the specific minor number range allocated belongs to any given caller. These minor numbers can be greater than 255. It is possible that the allocated range of device numbers could overflow the minor number range, spilling into the next major number. That behavior is enabled by design, and everything should work correctly - though, as far as your editor knows, no production kernel has any allocations which work that way. 

Regardless of which allocation function was used, device numbers can be returned to the system with: 

```
void unregister_chrdev_region(dev_t first, unsigned int count);
```

The association of device numbers with specific devices happens by way of the `cdev` structure, found in `<linux/cdev.h>`. It is possible to allocate an initialize a `cdev` structure with a sequence like: 

```
struct cdev *my_dev = cdev_alloc();
    
        if (my_dev != NULL)
        	my_dev->ops = &my_fops;  /* The file_operations structure */
    	my_dev->owner = THIS_MODULE;
        else
        	/* No memory, we lose */
```

In the more common usage pattern, however, the `cdev` structure will be embedded within some larger, device-specific structure, and it will be allocated with that structure. In this case, the function to initialize the `cdev` is: 

```
void cdev_init(struct cdev *cdev, const struct file_operations *fops);
        /* Need to set ->owner separately */
```

Either way, the structure is put into proper operating condition, and it will be equipped with the `file_operations` which should be invoked for the associated device. The `owner` field of the structure should be initialized to `THIS_MODULE` to protect against ill-advised module unloads while the device is active. 

The final step is to add the `cdev` to the system, associating it with the appropriate device number(s). The tool for that job is: 

```
int cdev_add(struct cdev *cdev, dev_t first, unsigned int count);
```

This function will add `cdev` to the system. It will service operations for the `count` device numbers starting with `first`; a `cdev` will often serve a single device number, but it does not have to be that way. Note that `cdev_add()` can fail; if the return code is zero, the device has _not_ been added to the system. 

Just as importantly: as soon as `cdev_add()` succeeds, the device is live, and its file operations can be called by the kernel. So a driver should not call `cdev_add()` until the initialization of the associated device is complete. To do otherwise is to invite unpleasant race conditions. 

Removal of a char device from the system is done with: 

```
void cdev_del(struct cdev *cdev);
```

The `cdev` should not be referenced after this call. In particular, if `cdev` was obtained with `cdev_alloc()`, it will likely be freed in `cdev_del()`. 

One final trick worth knowing about: when a char device's file operations are invoked, the associated `inode` pointer will be passed in, as usual. The field `inode->i_cdev` contains a pointer to the `cdev` structure for the device. Drivers can use that pointer to get to their own device-specific structure (perhaps with `container_of()`). It is, thus, no longer necessary to try to map the minor number onto an internal device - an operation which many drivers got wrong. 

The `cdev` interface evolved somewhat in early 2.6 releases, but has not seen any changes in some time.
