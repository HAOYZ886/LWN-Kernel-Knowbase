---
title: Netpoll is merged
url: https://lwn.net/Articles/75944/
date: "March 16, 2004"
category: Networking
author: 
---

One of the many new things merged into 2.6.5-rc1 is the "netpoll" infrastructure. Netpoll exists to support low-level kernel functions which may need to be able to send and receive packets over the network without involving the entire networking subsystem and without enabling interrupts. Examples include [kgdbeth](<https://lwn.net/Articles/48538/>) (which allows kernel debugging over the net), and netconsole, which enables remote, network-based consoles. The patches have been around (and in the -mm tree) for some time, but have only now found their way into the mainline. Netconsole was merged as well, but kgdbeth users will still have to apply patches for now. 

Supporting netconsole in network drivers turns out to be relatively easy - for most adaptors. There is a new `net_device` method called `poll_controller()`; its job is to catch up with whatever the device has been doing. For many devices, this method looks like this: 

```
static void poll_my_card(struct net_device *dev);
        {
    	disable_device_interrupts();
    	call_interrupt_handler(dev);
    	reenable_device_interrupts();
        }
```

Netpoll, in other words, is simulating device interrupts from within the kernel. Some device interrupt handlers may need tweaks to ensure that they do all of the necessary work without a real hardware interrupt, but most seem to work as they are.
