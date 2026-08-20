---
title: An API for specifying latency constraints
url: https://lwn.net/Articles/197299/
date: "August 28, 2006"
category: ACPI; Latency
author: 
---

Modern processors support a number of power states. When there is nothing of any real interest going on, they can be instructed to power down to one of potentially several different levels. Since processors on most systems are idle much of the time, this capability can be put to use to bring about a significant reduction in power use. Cutting power demand is most helpful on systems with limited power sources - laptops, portable music players, Linux-powered penguin robots, etc. - but cutting power consumption is also a good thing to do in most other environments as well. 

Powering down the CPU becomes an even more useful thing to do once a dynamic tick mechanism is in use - something which appears possible for the Linux i386 port in 2.6.19. The elimination of the periodic clock interrupt will allow the processor to sleep for longer periods of time when there is nothing to do. Longer sleeps can translate into deeper power saving modes, reducing consumption even further. 

The problem that can come up, however, is that the more aggressive power management modes will, by their nature, cause the processor to take longer to get back into an operating state. So, as the processor is put more deeply to rest, the system's latency in responding to external events will increase. In some situations, that latency can cause the system to fail to operate properly. Audio or video data might get dropped, a network adapter may start to see errors, or that robotic penguin could fail to respond in time to a cyber-walrus threat. The usual response to that problem, beyond hunting walruses to extinction, is to simply disable the power-saving behavior. but such drastic responses should not really be necessary. 

Various devices in the system, when operating in certain modes, will need to obtain responses from the system within a given period of time. The drivers for those devices know how the device is being operated at any given moment, so they know what the latency requirements are. If the system as a whole had that information, it could tune its operations to the minimum latency requirements in effect at the moment, and could change its operations as the requirements change. But there is no mechanism in the system for handling - and reacting to - this information. 

Arjan van de Ven has set out to change this situation with a [latency tracking infrastructure](<http://lwn.net/Articles/197282/>) patch. This work adds a set of new functions which may be used by drivers to indicate their latency requirements: 

```
#include <linux/latency.h>
    
        void set_acceptable_latency(char *identifier, int usecs);
        void modify_acceptable_latency(char *identifier, int usecs);
        void remove_acceptable_latency(char *identifier);
```

When a driver enters a mode where it has specific latency requirements (a camera driver starts acquiring frame data, say), it can tell the system about the maximum latency it can handle with `set_acceptable_latency()`. The `identifier` parameter is only used for identifying the request later on; `usecs` is the maximum latency in microseconds. The latency requirement can be changed with `modify_acceptable_latency()`, or eliminated altogether with `remove_acceptable_latency()`. 

The back end of the latency infrastructure includes a notifier chain for letting interested subsystems know when the maximum acceptable latency has changed. The current consumer of this information is the ACPI subsystem, which can use it to adjust the processor's idle state to meet that requirement. One could imagine that a smart dynamic tick implementation could use this information as well. 

In the current patch, only one subsystem (the IPW2100 wireless network driver) declares its latency requirements. This version of the patch has been proposed for inclusion in the -mm kernel, however, with the idea that other driver maintainers could start to make use of it. Unless some sort of surprising objection comes up, the latency management infrastructure looks likely to be a part of the 2.6.19 kernel.
