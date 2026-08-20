---
title: "Dealing with complexity: power domains and asymmetric multiprocessing"
url: https://lwn.net/Articles/449585/
date: "June 29, 2011"
category: Asymmetric multiprocessing; Power management
author: "By Jonathan Corbet June 29, 2011"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
June 29, 2011

When one thinks of embedded systems, it may be natural to think of extremely simple processors which are just barely able to perform the tasks which are asked of them. That view is somewhat dated, though. Contemporary processors are often put into settings where they are expected to manage a variety of peripherals - cameras, signal processors, radios, etc. - while using a minimum of power. Indeed, a reasonably capable system-on-chip (SoC) processor likely has controllers for these peripherals built in. The result is a processor which presents a high level of complexity to the operating system. This article will look at a couple of patch sets which show how the kernel is changing to deal with these processors. 

#### Power domains

On a desktop (or laptop) system, power management is usually a matter of putting the entire CPU into a low-power state when the load on the system allows. Embedded processors are a little different: as noted above, they tend to contain a wide variety of functional units. Each of these units can be powered down (and have its clocks turned off) when it is not needed, while the rest of the processor continues to function. The kernel can handle the powering down of individual subsystems now; what makes things harder is the power dependencies between devices. 

Power management was one of the motivations behind the addition of the kernel's device model in the 2.5 development series. It does not make sense, for example, to power down a USB controller if devices attached to that controller remain in operation. The device model captures the connection topology of the system; this information can be used to power devices up and down in a reasonable order. The result was much improved power management in the 2.6 kernel. 

On newer systems, though, there are likely to be dependencies between subsystems that are not visible in the bus topology. A set of otherwise unrelated devices may share the same clock or power lines, meaning that they can only be powered up or down as a group. Different SoC designs may feature combinations of the same controllers with different power connections. As a result, drivers for specific controllers often cannot know whether it is safe to power down their devices - or even how to do it. This information must be maintained at a level separate from the device hierarchy if the system is to be able to make reasonable power management decisions. 

The answer to this problem would appear to be Rafael Wysocki's [generic I/O power domains](<https://lwn.net/Articles/449302/>) patch set. A power domain looks like this: 

```
struct generic_pm_domain {
    	struct dev_pm_domain domain;	
    	struct list_head sd_node;	
    	struct generic_pm_domain *parent;
    	struct list_head sd_list;
    	struct list_head dev_list;
    	bool power_is_off;
    	int (*power_off)(struct generic_pm_domain *domain);
    	int (*power_on)(struct generic_pm_domain *domain);
    	int (*start_device)(struct device *dev);
    	int (*stop_device)(struct device *dev);
            /* Others omitted */
        };
```

Power domains are hierarchical, though the hierarchy may differ from the bus hierarchy. So each power domain has a parent domain (`parent`), a list of sibling domains (`sd_node`), and a list of child domains (`sd_list`); there is also, naturally, a list of devices contained within the domain (`dev_list`). When the kernel is changing a domain's power state, it can use `start_device()` and `stop_device()` to operate on specific devices, or `power_on()` and `power_off()` to power the entire domain up and down. 

That is the core of the patch though, naturally, there is a lot of supporting infrastructure to manage domains, let them participate in suspend and resume, etc. The one other piece is the construction of the domain hierarchy itself. The patch set includes one example implementation which is added to the ARM "shmobile" subarchitecture board file. In the longer term, there will need to be a way to represent power domains within device trees since board files are intended to go away. 

This patch set has been through several revisions and seems likely to be merged during the 3.1 development cycle. 

#### Asymmetric multiprocessing

When one speaks of multiprocessor systems, the context is almost always symmetric multiprocessing - SMP - where all of the processors are equal. An embedded SoC may not be organized that way, though. Consider, for example, this description from the introduction to [a patch set from Ohad Ben-Cohen](<https://lwn.net/Articles/448562/>): 

OMAP4, for example, has dual Cortex-A9, dual Cortex-M3 and a C64x+ DSP. Typically, the dual cortex-A9 is running Linux in a SMP configuration, and each of the other three cores (two M3 cores and a DSP) is running its own instance of RTOS in an AMP configuration. 

Asymmetric multiprocessing (AMP) is what you get when a system consists of unequal processors running different operating systems. It could be thought of as a form of (very) local-area networking, but all of those cores sit on the same die and share access to memory, I/O controllers, and more. This type of processor isn't simply "running Linux"; instead, it has Linux running on some processors trying to shepherd a mixed collection of operating systems on a variety of CPUs. 

Ohad's patch is an attempt to create a structure within which Linux can direct a processor of this type. It starts with a framework called "remoteproc" that allows the registration of "remote" processors. Through this framework, the kernel can power those processors up and down and manage the loading of firmware for them to run. Much of this code is necessarily processor-specific, but the framework abstracts away the details and allows the kernel to deal with remote processors in a more generic fashion. 

Once the remote processor is running, the kernel needs to be able to communicate with it. To that end, the patch set creates the concept of "channels" which can be used to pass messages between processors. These messages go through a ring buffer stored in memory visible to both processors; [virtio](<https://lwn.net/Articles/239238/>) is used to implement the rings. A small piece of processor-specific code is needed to implement a doorbell to inform processors of when a message arrives; the rest should be mostly independent of the actual system that it is running on. 

This patch set has been reasonably well received as a good start toward the goal of regularizing the management of AMP systems. A complete solution is likely to require quite a bit more work, including implementations for a wider variety of architectures. But, then, one could say that, after twenty years, Linux as a whole is still working toward a complete solution. The hardware continues to evolve toward more complexity; the operating system will have to keep evolving in response. These two patch sets give some hints of the direction that evolution is likely to take in the near future.
