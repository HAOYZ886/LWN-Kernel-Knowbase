---
title: Linux and wireless networking
url: https://lwn.net/Articles/167270/
date: "January 11, 2006"
category: "Networking-Wireless; Wireless extensions"
author: 
---

Jeff Garzik's recent [State of the Union: Wireless](<http://lwn.net/Articles/166602/>) posting came right to the point: 

Another banner year has passed, with Linux once again proving its superiority in the area of crappy wireless (WiFi) support. Linux oldsters love the current state of wireless, because it hearkens back to the heady days of Yuri Gagarin, Sputnik and Linux kernel 0.99, when getting hardware to work under Linux required either engineering knowledge or luck (or both). 

Jeff went on to discuss a few of the challenges facing the Linux wireless implementation. This is, indeed, one area where some real progress is needed. Proprietary chipsets are just the beginning of the issues which must be dealt with - free software developers are actually beginning to catch up in that area. But before all the resulting drivers can be merged into a coherent whole, a few other things will have to be worked out. 

One of those has to do with the 802.11 stack used by the kernel. As was [discussed here last December](<http://lwn.net/Articles/162890/>), there is a fair amount of unhappiness with the in-kernel stack, which, among other things, has no "softmac" support, needed for adapters which do not perform MAC functions in hardware. A number of out-of-tree wireless stacks do provide that support, and there have been a lot of suggestions that one of those (usually the [DeviceScape stack](<http://kernel.org/pub/linux/kernel/people/jbenc/>)) be merged. 

Those suggestions have been strongly resisted by the networking maintainers. They would rather see work go into fixing up the stack which is in the kernel now than replace it wholesale or - even worse - having two independent 802.11 stacks to maintain. Replacing the current stack would involve significant disruption in the networking subsystem, and would be hard to do without breaking the drivers which use the old stack. The two-stack solution, instead, would bloat the kernel and increase the amount of work required to maintain the networking subsystem into the future. So it is not surprising that there is a strong interest in evolving the current stack toward the desired functionality rather than bringing in a whole new implementation. 

Still, the pressure to switch over to the DeviceScape stack appears to be growing. Jeff's posting seems to recognize this fact, and asks that, in the end, the developers at least pick a single stack which they can live with. And, says Jeff, regardless of which stack is chosen in the end: 

It is currently fashionable to laud DeviceScape and trash in-kernel ieee80211, but outside of the cheerleading, BOTH have real technical issues that need addressing. IOW, no matter what code is chosen, _somebody_ is on the hook for a fair amount of work. A switch is not without its costs. 

Another issue has to do with the management interface for wireless adapters. Wired network adapters are relatively simple; set a few options on media access, give them an address, and they are ready to go. The wireless world is rather more complicated. To deal with the extra configuration required by wireless adapters, the "wireless extensions" interface - essentially a big set of `ioctl()` commands for querying and setting adapter parameters - was developed. 

There seems to be a consensus that the wireless extensions have reached their expiration date, and need to be replaced with something else. Most developers would appear to favor a new (not yet specified) interface built on the netlink mechanism. User-space management code could then be rewritten to speak the new management protocol over netlink sockets. 

This approach may seem strange, given the emphasis which has been placed on sysfs and the creation of scriptable, plain-text interfaces. Sysfs does seem like a poor match for wireless configuration, however. Wireless adapters have a large number of parameters, and it is often necessary to change several of them simultaneously. Sysfs, with its one-value-per-file rules, provides no means for this sort of atomic, multi-parameter update; a netlink interface could, instead, be designed with these needs in mind from the beginning. 

Of the other issues mentioned, perhaps this one is the most significant: there is no wireless maintainer. The lack of a developer who is specifically interested in this area of networking and who will work to push it forward has clearly hurt. Fortunately, it appears that this era may be at an end: John Linville has [stepped forward](<https://lwn.net/Articles/167272/>) to take on this responsibility. 

John has a fair amount of work ahead of him; quite a few developers have to be brought together and made to agree on the way forward. To that end, a wireless networking summit has been [scheduled](<http://developer.osdl.org/shemminger/blog/?p=29>) for early April in Portland. If the attendees at that meeting (which looks to include both kernel and user space developers) can produce a viable plan, Linux may just lose its "superiority in the area of crappy wireless support" before too long.
