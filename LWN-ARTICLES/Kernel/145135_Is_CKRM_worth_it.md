---
title: "Is CKRM worth it?"
url: https://lwn.net/Articles/145135/
date: "July 27, 2005"
category: "Class-based resource management"
author: 
---

Anytime your editor gives a talk on kernel development efforts, there seems to be one project which inspires scattered boos and hisses from the audience. The lucky project this year was Class-based Kernel Resource Management (CKRM). The CKRM patches have been under development for some time, and the developers involved have been pushing for inclusion. The future of the CKRM patches seems uncertain, however; there is significant opposition to them being merged. 

The idea behind CKRM is to give system administrators a high degree of control over how the resources on a system are used. To that end, it puts every process into a "class," then applies rules specifying which resources are available to each class. On the classification side, CKRM includes a rule-based classification engine which can pigeonhole processes in a number of ways: its user or group IDs, the command it is running, which ports it is listening to, etc. Classification engines are pluggable, however, so a site with specific needs could write its own. It is also possible for an administrator to directly shove a process into a given class by way of a virtual filesystem interface. 

The controlling side regulates how much of the system each class can use. Maximum limits can be applied, in a way similar to the resource limits built into the kernel now. There is also a mechanism for specifying a "guarantee," a minimum amount of resource which will be allocated to a class. So an administrator can set things up such that the web server will not take more than half the CPU, or that the X server will always get at least 20% if it needs it. 

That leads to another component of CKRM: controllers. Each controller manages the allocation of one specific resource in the system. CPU usage is regulated by the CPU controller; as it happens, the CKRM patches in the -mm tree do not currently include that controller. The CPU controller extends its fingers fairly deeply into the Linux scheduler, and the developers do not feel that it is ready for inclusion quite yet. In fact, the only controllers currently in -mm handle the total number of tasks and the rate at which processes can fork. Many other controllers are in development, handling resources like main memory, disk I/O bandwidth, network bandwidth, and more. 

The CKRM patches are large - over 14,000 lines in -mm. They also must place hooks into many sensitive parts of the kernel in order to be able to monitor process transitions and enforce resource limits and guarantees. Any patch which digs into parts of the core kernel in this way is going to see a fair amount of scrutiny, and CKRM is no exception. In this case, many developers see CKRM as an overly complex subsystem which is aimed at the needs of the customers of one specific vendor. Most Linux users simply do not need to have such fine-grained control over resource usage on their systems. 

CKRM looks like a bit of a long-term maintenance headache as well. Every subsystem which requires distributing hooks around the kernel (think of the Linux security modules, the audit subsystem, or inotify as other subsystems of this type) is essentially overlaying a new structure on top of the base kernel. Any changes to the kernel must be done carefully so that none of the overlaid structures will break. So each one of these structures makes kernel programming a little harder; it is one more thing a developer must keep in mind when making changes. Mix in the fact that most kernel developers (and testers, for that matter) will not have CKRM configured into their kernels, and it becomes clear that a subsystem like CKRM could turn out to be relatively fragile. 

Supporters of CKRM see it as a useful tool for the management of larger systems (they see applications for smaller systems as well). In particular, it can be used with virtualization systems (Xen or UML, for example) to consolidate servers onto a smaller hardware base while providing appropriate resource guarantees for the guest servers. Thus, [says Gerrit Huizenga](<https://lwn.net/Articles/145159/>), CKRM can be thought of as part of the "eco-computing movement." CKRM imposes no overhead on the system if it is configured out, and almost no overhead if it is built in but not used. Since CKRM is useful for some users, and stays out of the way for the rest, it is worth adding to the kernel. 

For now, CKRM is in -mm for people to play with; Andrew Morton has noted that it is not, yet, on a path toward inclusion in the near future. He [wants to see a real debate](<https://lwn.net/Articles/145160/>), however, and not a simple, offhand rejection: 

But there's been a lot of work put into this and if we're to flatly reject the feature then the developers are owed a much better reason than "eww yuk". 

So far, that reason has not been provided in any definitive way. So expect to see this topic come up again as the developers try to get a real answer on whether CKRM is headed for the mainline or not.
