---
title: Runtime power management
url: https://lwn.net/Articles/347573/
date: "August 19, 2009"
category: Power management
author: "By Jonathan Corbet August 19, 2009"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
August 19, 2009

While a great deal of power management work has been done on Linux systems in recent years, much of that work has been directed toward the creation of working suspend and hibernation capabilities. But there is more to power management than that; there is real value in being able to minimize the power consumption of a running system. That is as true for large data center machines as it is for laptops; reduced power usage and lower air conditioning requirements have both economic and environmental benefits. Now that the suspend problem is mostly solved, increasing amounts of attention are being paid to the other aspects of power management; some recent patches show how the infrastructure for runtime power management is coming together. 

The core of the future power management structure appears certain to be Rafael Wysocki's [runtime power management patch](<https://lwn.net/Articles/347574/>). This patch set adds structure to the power management code to facilitate the suspending and resuming of individual system components at run time. The `dev_pm_ops` structure is augmented with three new functions: 

```
int (*runtime_suspend)(struct device *dev);
        int (*runtime_resume)(struct device *dev);
        int (*runtime_idle)(struct device *dev);
```

These functions are to be implemented by the core device code for each bus type; they may then be turned into bus-specific driver callbacks. The power management code will call `runtime_suspend()` to prepare a specific device for a lower-power state. This call does not imply that the device itself must suspend, but the device does need to prepare for a condition where it is no longer able to communicate with the CPU or memory. In other words, even if the device does not suspend, hardware between that device and the rest of the system might suspend. A return value of `-EBUSY` or `-EAGAIN` will abort the suspend operation. 

A call to `runtime_resume()` should prepare the device to operate again with the rest of the system; the driver should power up the device, restore registers, and do anything else needed to get the device functioning again. The `runtime_idle()` callback, instead, is called when the core thinks that the device is idle and might be a good candidate for suspending. The callback should decide whether the device can really be suspended (this could include checking the state of any other devices connected to it) and, if all seems well, initiate a suspend operation. 

Along with these callbacks is, of course, a set of core functions designed to manage suspend and resume activities, deal with mid-course cancellations, allow outside code to make power management changes, and so on. See [the associated document file](<https://lwn.net/Articles/347575/>) for more information on how this subsystem works. 

The code described above has been through several review iterations and would appear to be on track for merging in 2.6.32. Rafael's [asynchronous suspend and resume patch](<http://lwn.net/Articles/346767/>), instead, is rather newer and may take a little longer. The idea behind this patch is to extend the runtime power management code to allow suspend and resume callbacks to be invoked asynchronously; that, in turn, would allow them to be run in parallel. As long as there are no dependencies between a pair of devices, suspending or resuming them in parallel should make full-system transitions faster. 

The problem is in the dependencies; running a bunch of power management operations in parallel increases the risk of getting the order wrong. To avoid this outcome, the patch adds a new completion object to each device; when a device is to be suspended, the completions will be used to ensure that any dependent devices are suspended first. At resume time the completions are used in the reverse direction: devices wait for their parent devices to be resumed before resuming themselves. As long as the dependency information is correct, this mechanism should ensure that a set of power management threads can run in parallel without breaking the system. 

Ensuring that the dependencies are correct was one of the reasons for the creation of the Linux device model years ago. With a properly-constructed tree, the system can know, for example, that it cannot suspend a USB controller until all USB devices plugged into it have been suspended. In turn, the PCI controller to which the USB controller is attached must remain functional until the USB controller is suspended, and so on. The problem is that system dependencies are not always that simple. A PCI device may also require the services of an I2C controller, for example, or devices can be combined on multi-function chips in surprising ways. So the device tree has proved unable to represent all of the power management dependencies in the system. 

Rafael has addressed this problem in a later version of the patch, which adds a new framework for representing power management dependencies. At the core of it is this structure: 

```
struct pm_link {
        	struct device *master;
        	struct list_head master_hook;
        	struct device *slave;
        	struct list_head slave_hook;
        };
```

One of these structures exists for each dependency known to the system. It indicates that the "master" device must always be functional whenever the "slave" device is; the master must be suspended after the slave and resumed before it. Many of these links can be created by the power management core itself; others will have to be generated by the relevant drivers. There have been some [concerns](<https://lwn.net/Articles/347580/>) raised about the memory use of this structure, but a better solution has not yet been proposed. 

Meanwhile, Matthew Garrett has taken the core power management code one step further with [a set of runtime power management patches](<http://lwn.net/Articles/346898/>) of his own. He has pushed the new power management calls down into the PCI and USB bus layers and used them to suspend devices opportunistically as the system runs; he reports a power savings of around 0.2 watts as a result. Review comments resulted in these patches being withdrawn for now for repairs, but they show the direction things are heading. With sufficient software support and cooperative hardware, Linux should be able to further reduce the operating power needed for whole classes of systems. That cannot fail to be a good thing.
