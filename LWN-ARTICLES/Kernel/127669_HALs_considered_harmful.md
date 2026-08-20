---
title: HALs considered harmful
url: https://lwn.net/Articles/127669/
date: "March 15, 2005"
category: Device drivers; Hardware abstraction layers
author: 
---

It is a nice thing when hardware vendors provide Linux drivers for their products. Since these drivers are written by the vendor, there is usually no trouble getting information on how the hardware is controlled. With luck, that hardware will "just work" for Linux users, and all will be as it should be. In the real world, however, things are not always that simple. Hardware companies often take interesting approaches to coding drivers, and the people involved are not always well tied into the Linux kernel development community. The result can be conflicts between the vendors, who simply want to get things done, and the kernel developers, who are increasingly unwilling to accept code which does not meet their standards. 

For a current example, consider the [proposed new Neterion/S2io 10GbE network driver](<https://lwn.net/Articles/127532/>). This driver has been rewritten from the beginning; it supports many of the hardware's advanced features and provides high performance. It looks like just the thing for high-end Linux-based networking uses. 

The problem is that the driver does not deal directly with the Linux kernel API. It is, instead, based on a "hardware abstraction layer" (HAL) which glues the driver to the kernel. So, for example, the driver builds lists with a structure like: 

```
typedef struct xge_list_t {
    	struct xge_list_t* prev;
    	struct xge_list_t* next;
        } xge_list_t;
```

Such lists are accessed with functions like `xge_list_insert()` and even `xge_list_for_each()`. Similarly, the driver uses `xge_os_spin_lock()` to acquire a lock, `xge_os_malloc()` to allocate memory, and `xge_os_pio_mem_read8()` to read a byte from I/O memory. This approach helps Neterion support a variety of systems with the same core driver code, but it does not sit well with the kernel hackers. Networking maintainer David Miller [responded](<https://lwn.net/Articles/127677/>) this way: 

I totally reject this driver, HAL is unacceptable for in-tree drivers. We've been over this a thousand times. 

One problem with the HAL approach is that there can be a performance cost. A 10G network adaptor can handle thousands of packets per second; at that sort of load, even the minimal overhead of a simple wrapper function can make a significant difference. The extra memory taken by the glue code, parallel linked list implementation, etc. also hurts. A developer community which is dedicated to obtaining the best possible performance from the hardware will be unwilling to swallow even a small cost in the name of portability. 

The bigger issue, however, is in the maintainability of the driver. A driver written for a HAL layer has its own idioms and conventions; it works with a completely different API. It simply does not look like a Linux driver; Linux developers will have a harder time understanding and modifying it. One might think that this is not a big issue, since Neterion has said that it plans to maintain the driver, but there are a couple of problems that come up: 

  * When a kernel developer changes an internal function, he or she will usually go through and fix all of the in-tree users of that function. So developers who are not employed by the hardware vendor will almost certainly have to work with the driver code at some point. 

  * Hardware vendors have a short attention span. Product cycles tend to be short, and the vendor will, before too long, move on to new products requiring new and different drivers. Once a given driver no longer applies to the products which are currently in the vendor's catalog, the vendor will, most likely, see little reason to continue maintaining that driver. The Linux community, however, will have an interest in keeping that driver working for several more years. 

Additionally, the vendor may resist patches which affect the HAL layer itself, making it harder for the community to work on the driver. Overall, the Linux kernel developers plan to maintain the kernel for many years into the future; they tend to be concerned about taking on code which will make that maintenance task harder in the future. 

So the kernel hackers have some solid reasons for resisting HAL-based drivers. The vendors also have good reasons for wanting to write such drivers. To them, the resistance to HAL looks like a "Linux is the only important system" attitude, and it forces them in incur extra costs when writing their code. In this case, Neterion has [reluctantly said](<https://lwn.net/Articles/127680/>) that it will produce a non-HAL driver if that is the only way to get into the tree; other vendors may not bother.
