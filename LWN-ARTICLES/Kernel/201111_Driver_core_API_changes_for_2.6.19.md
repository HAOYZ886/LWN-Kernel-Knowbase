---
title: Driver core API changes for 2.6.19
url: https://lwn.net/Articles/201111/
date: "September 26, 2006"
category: "Device drivers-Support APIs; Device model; Power management"
author: 
---

The Linux driver core subsystem continues to evolve at a high rate. The [set of patches for 2.6.19](<http://lwn.net/Articles/201100/>) continues this process with a number of improvements - and a number of API changes. This time around, however, the changes appear to be additive, and thus should not break any existing drivers. 

Linux boot time is an ongoing sore point - there are few users who wish that their systems would take longer to come up. There are many things which happen during the boot process, and many possible ways of speeding things up. Most of the opportunities for improving boot time lie in user space, but, on the kernel side, probing for devices can take a lot of time. Each device must be located, initialized, and hooked into the system; this process can involve waiting for peripheral processors to boot, firmware loads, and, perhaps, even physical processes like spinning up disks. As a result, much of the kernel time spent bringing up devices is idle time, waiting for the device to do its part. 

One obvious idea for improving this process is to probe devices in parallel. That way, when the kernel is waiting for one device to respond, it can be setting up another; the kernel would also be able to make full use of multiprocessor systems. The 2.6.19 device core will, at last, have the ability to operate in this mode. The changes start by adding a flag (`multithread_probe`) to the `device_driver` structure. At probe time, if a driver has set that flag, the actual work of setting up the device will be pushed into a separate kernel thread which can run in parallel with all the others. At the end of the initialization process, the kernel waits for all outstanding probe threads to finish before mounting the root filesystem and starting up user space. 

On uniprocessor systems, this change leads to a relatively small reduction of bootstrap time. Drivers typically do not yield the processor during the probe process, so there is relatively little opportunity for parallelism, even during times when the kernel has to wait for a bit. On multiprocessor systems, however, the effect can be rather more pronounced - each CPU can be probing devices in parallel with all the others. So this change will be most useful on large systems with lots of attached devices. 

At least, it will be useful once it's enabled; this feature is currently marked "experimental" and carries a number of warnings. Even when it is turned on, it only applies to PCI devices. Not all drivers are written with parallel probing in mind, so they may not have the right sort of locking in place. There can be problems with power drain - turning on too many devices simultaneously can cause a high demand for power over a short period of time; if this demand exceeds what the power supply can deliver, the resulting conflagration could slow the boot process considerably. The order of device enumeration is likely to become less deterministic. And so on. Still, this feature, over time, should lead to faster system boots, especially on systems (such as embedded applications) where the mix of hardware is well understood and static. 

On a separate front, the API for handling suspend and resume has been filled out somewhat. The class mechanism now has its own hooks, found in `struct class`: 

```
int (*suspend)(struct device *dev, pm_message_t state);
        int (*resume)(struct device *dev);
```

The new `suspend()` method is called relatively early in the suspend process, and is expected to handle any class-specific tasks. These might include quieting the device and stopping higher-level processing. The `resume()` method is called toward the end of the resume process and should finish the job of getting devices in the class ready to operate again. 

Most of the suspend/resume processing is still handled through the bus subsystem, however. That portion of the API has been filled out with three new `struct bus_type` methods: 

```
int (*suspend_prepare)(struct device *dev, pm_message_t state);
        int (*suspend_late)(struct device *dev, pm_message_t state);
        int (*resume_early)(struct device *dev);
```

All of these methods just add more places for the bus code to hook into the process and do whatever work needs to be done. So `suspend_prepare()` is called early on, while the system is still in an operational state. The `suspend()` method is unchanged from prior kernels: it is called after tasks have been frozen, and is allowed to sleep if need be. The new `suspend_late()` method, instead, is called very late, with interrupts disabled and only a single processor running. At resume time, `resume_early()` is called, once again, with interrupts and SMP disabled, and the old `resume()` method is called later. 

The PCI subsystem makes this new functionality available via three new methods in the `pci_driver` structure: 

```
int  (*suspend_prepare) (struct pci_dev *dev, pm_message_t state);
        int  (*suspend_late) (struct pci_dev *dev, pm_message_t state);
        int  (*resume_early) (struct pci_dev *dev);
```

There are no drivers actually using these new methods in the mainline, as of this writing. 

Finally, the class subsystem continues to migrate toward the eventual removal of the `class_device` structure. To that end, `struct class` has picked up another pair of methods: 

```
int (*dev_uevent)(struct device *dev, char **envp, int num_envp,
    		      char *buffer, int buffer_size);
        void (*dev_release)(struct device *dev);
```

These methods provide similar functionality as the `uevent()` and `release` methods in `struct class_device`. 

Also as part of this migration, a couple of new helper functions have been added: 

```
int device_create_bin_file(struct device *dev, 
                                   struct bin_attribute *attr);
        void device_remove_bin_file(struct device *dev, 
                                    struct bin_attribute *attr);
```

This methods will let drivers create binary attributes in sysfs without having to deal with the sysfs code directly.
