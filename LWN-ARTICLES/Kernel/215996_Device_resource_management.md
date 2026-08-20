---
title: Device resource management
url: https://lwn.net/Articles/215996/
date: "January 2, 2007"
category: Development tools; IO memory; Resources
author: 
---

Writing device drivers can be a tricky task. Simply getting a piece of hardware to operate as desired - perhaps working from erroneous or nonexistent documentation - can be a frustrating process. Beyond that, however, the driver must allocate several different types of resources for the device; these resources can include I/O memory mappings, interrupt lines, blocks of memory, DMA buffers, registrations with multiple subsystems, etc. All of these allocations must be returned to the system when the device (or its driver) goes away. It is not uncommon for driver writers to forget to deallocate something, leading to resource leaks. 

The problem can get worse, however, in the face of initialization errors. If the driver is unable to properly set up its device, it must undo any registrations which had been done up to the point of failure. Attempts to handle initialization failures usually take the form of several `goto` labels within the initialization function or some sort of global "initialization state" variable describing where cleanup should begin. Either way, these paths tend not to be well tested, so the chances of an initialization failure leading to some sort of resource leak are quite good. 

Tejun Heo, who has done much to improve the Linux serial ATA subsystem over the last year, has had enough of these sorts of initialization problems. So he has put together [a device resource management patch](<http://lwn.net/Articles/215861/>) which, if accepted, has the potential to make driver code simpler and more robust. The core idea is simple: every time a driver allocates a resource, the management code remembers the allocation and any information needed to free that allocation. When the driver disconnects from the device, all of the remembered allocations are returned to the system. 

This sort of allocation tracking cannot be added to the current API in any sort of coherent way. Tejun's patch, instead, creates new "managed" versions of various allocation functions. The new functions look like the old ones with (1) the addition of "`m`" (or "`devm`") to the name, and (2) a `struct device` argument if the function did not already have one. So, for example, the managed versions of the interrupt allocation functions are: 

```
int devm_request_irq(struct device *dev, unsigned int irq,
    		         irq_handler_t handler, unsigned long irqflags,
    		    	 const char *devname, void *dev_id);
        void devm_free_irq(struct device *dev, unsigned int irq, 
                           void *dev_id);
```

The patch also includes managed functions for dealing with DMA buffers, I/O memory regions, plain memory allocations, and PCI device setup. They allow the driver author to replace a whole set of deallocation calls with a simple call to `devres_release_all()`, simplifying the code significantly. In fact, even that call is unnecessary; the driver core will call it when the driver detaches from the device. 

For more complicated situations, there is also a "group" concept. Groups can be thought of as markers in the stream of allocations associated with a given device. The allocations performed within a specific group can be rolled back without affecting any others. In brief, the group API is: 

```
void *devres_open_group(struct device *dev, void *id, gfp_t gfp);
        void devres_close_group(struct device *dev, void *id);
        void devres_remove_group(struct device *dev, void *id);
        int devres_release_group(struct device *dev, void *id);
```

A call to `devres_open_group()` will create a new group for the given device, identified by the `id` value. Any allocations performed thereafter will be considered to be a part of that group until `devres_close_group()` is called. If initialization works as desired, however, `devres_remove_group()` can be used to get rid of the group overhead while leaving the allocations (and their tracking information) untouched. In the failure path, `devres_release_group()` will return all allocations belonging to the given group. 

There has been very little discussion of this patch set, as of this writing. Driver writers, perhaps, are still recovering from the holiday festivities. It is not too hard to imagine that there could be some discomfort about the extra overhead involved in tracking all of those allocations - especially since things do function normally almost all of the time. In the end, however, the promise of correct operation in a wider range of situations may be enough to motivate the inclusion of the new interface.
