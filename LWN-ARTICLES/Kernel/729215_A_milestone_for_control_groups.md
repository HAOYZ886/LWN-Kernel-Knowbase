---
title: A milestone for control groups
url: https://lwn.net/Articles/729215/
date: "July 31, 2017"
category: "Control groups-Thread-level control"
author: "By Jonathan Corbet July 31, 2017"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
July 31, 2017

Changes to core-kernel subsystems take time but, even so, one can only imagine that Tejun Heo never expected the process of fixing the control-group interface to take more than five years. Disagreements over the design of the new control-group interface have delayed its adoption; even though most of the code has been in the kernel for some time, not all controllers work with it. It would now appear, however, that agreement has been reached on an important final piece, which is currently on track to be merged for the 4.14 development cycle. 

When Heo first [raised the issue](<https://lwn.net/Articles/484251/>) of fixing the control-group interface in 2012, he identified what he saw as two key problems: the ability to create multiple control-group hierarchies and allowing a control group to contain both processes and other control groups. Both interface features complicated the implementation of controllers, especially in cases where multiple controllers need to be able to cooperate with each other. His proposal was that the new ("V2") control-group API should dispense with these features. 

Fast-forward to 2017, and those changes have been made. The V2 interface supports a single control-group hierarchy, and it requires that processes only appear in the leaf nodes of that hierarchy. Getting there took quite a bit of discussion and negotiation, and most users have made their peace with the new world order. This migration [ran into a snag](<https://lwn.net/Articles/697366/>) when the time came to update the CPU controller, though, with the result that there still is no CPU controller for the V2 interface. 

The core problem is the "no internal processes" rule, combined with another V2 constraint that was added a bit later: all of the threads of any given process must be placed in the same control group. For most of the controllers in the system, it makes little sense to place a process's threads in different parts of the hierarchy; many resources are best managed at the process level. But CPU scheduling is different. It is entirely sensible (and useful) to allow a thread to compete with a subgroup full of other threads for the CPU, and applying different scheduling constraints to different threads in the same process is also useful. The inability of the V2 interface to handle this use case has led to disagreements that have taken years to resolve. 

Heo has made various proposals to address this problem, culminating in the "[thread mode" concept](<https://lwn.net/Articles/715051/>) posted in February. There were still some disagreements at that time that prevented thread mode from being merged, but it would appear that those have, finally, been worked out. 

#### Thread mode for 4.14

The thread-mode concept found in [the latest patch set](<https://lwn.net/Articles/729219/>) follows the same lines as the version described in February. In current kernels, all control groups adhere to the "no internal processes" and "all of a process's threads are grouped together" rules. Control groups following these rules still exist in the new scheme; indeed, that remains the default mode. Such groups have been deemed "domain groups". 

A domain group can be changed to a "threaded group" by writing the string "`threaded`" to its `cgroup.type` control file. The group must be empty for this change to be allowed. Threaded groups differ from domain groups in a few ways: 

  * Any subgroups of a threaded group must also be threaded groups. Interestingly, new groups under a threaded group start out as domain groups in an "invalid" and unusable state. The only thing that can be done with them (other than removal) is to switch them to the threaded mode. 

  * The peers of a threaded group must also be threaded groups. In other words, a domain group that contains a threaded group can only contain threaded groups. An attempt to create a domain group inside a group that contains threaded groups will yield a group in the "invalid" state. 

  * The "no internal processes" rule does not apply within threaded groups; a threaded control group can contain both processes and other threaded control groups. 

  * The requirement that all of a process's threads must be in the same group is also relaxed. Those threads may now be placed in multiple groups, but all of those groups must be threaded and a part of the same hierarchy. 

As an example, consider the hierarchy from the February article shown on the right. Here, "A" and "B" are traditional domain groups, while "T1" and "T2" are a pair of threaded control groups. T1 violates the "no internal ![\[Control-group hierarchy\]](https://static.lwn.net/images/2017/cgtm2.png) processes" rule because it contains both T2 and the process P3, but, since it's a threaded group, that configuration is allowed. It is also legal for P2 and P3 to be threads of the same process. These aspects of the hierarchy are not possible without the new threaded group concept. 

A resource controller that is not aware of threaded groups will not see them at all. Consider the memory controller, for example, which is hard to implement in a rational way in the threaded mode. That controller will see P2 and P3 as being contained within the domain group B; the internal hierarchy will be hidden from it. The rules against internal processes and distributed threads still exist for such a controller. 

On the other hand, a controller that is able to handle threaded groups can indicate that fact to the kernel, and it will have the full hierarchy available to it. These controllers must have a sensible concept for what it means to have processes competing against groups for resources, and they must be able to apply different policies to threads belonging to the same process. Some resources are not amenable to control in that mode, but others work well. The patch enabled threaded mode for the PID and perf_events controllers, neither of which needed changes beyond setting the requisite flag. Interestingly, the CPU controller has not yet been enabled with the new interface; that is a bigger job that may be waiting for the current patch set to be merged. 

One significant difference from the February patch set is the establishment of a special rule for the root control group. That group was already unique in that it was exempt from the "no internal processes" rule; it is also uniquely able to contain both threaded and domain groups. This exemption was added to allow performance-sensitive threaded groups to be placed as high as possible in the hierarchy. Placing tasks lower in the hierarchy adds a bit of overhead that, while small, is unwelcome to those trying to squeeze every drop of performance out of their systems. 

Having finally managed to address all of the objections, Heo [announced](<https://lwn.net/Articles/729220/>) on July 21 that the threaded mode had been queued for merging in 4.14. Without the CPU controller this merging doesn't quite mark the end of the V2 conversion, but that end is now at least in sight. 

#### Bypass mode

Of course, the "completion" of the V2 interface does not mean that the work is actually done; few things in the kernel are ever truly finished. Developers are already thinking about ways in which this interface could be extended to accommodate other use cases. One such extension is the "[bypass mode](<https://lwn.net/Articles/728621/>)" proposed by Waiman Long. 

Resource distribution in control groups is a top-down matter: a controller can only be enabled for a group if it's enabled in that group's parent. If one looks at the simple control-group hierarchy to the right, for ![\[Control
group hierarchy\]](https://static.lwn.net/images/2017/cgroup-hier.png) example, it is only possible to enable any given controller in group C if it has already been enabled in group A. That is not usually a problem but, Long says, there may be situations where the requirement to enable the controller in group A gets in the way. The above-mentioned issue with scheduler performance maybe one such case: enabling the CPU controller in A will result in a small performance penalty for group C. 

To enable more flexibility in how controllers see the hierarchy, Long's patch set adds a new "bypass" mode. This mode disables a controller in the group for which it is set, but still allows the controller to be enabled further down the hierarchy. So, in this case, the controller could be set to bypass group A, but to be enabled in group C. For all practical purposes, bypass mode would simply hide group A from the bypassed controller, changing its view of the hierarchy. 

Heo's [response](<https://lwn.net/Articles/729222/>) to this patch set is that bypass mode ""continues to be an interesting idea"", but the changes are intrusive and he would like to see some serious use cases first. Long [described some uses](<https://lwn.net/Articles/729223/>) in further detail, but the conversation has not progressed much beyond that point. So while something like bypass mode may eventually become a part of the control-group API, it is probably not likely to happen in the immediate future. 

In a more general sense, though, the control-group API finally appears to be getting close to the point that was envisioned over five years ago when this effort began. The new API is near to its intended functionality, and the major design disagreements seem to have been worked out. There will, doubtless, be plenty of room for new features (and arguments associated with them) for a long time, and there is still the issue of someday phasing out the V1 interface. But control-group development is reaching an important milestone and, with luck, things will be a bit calmer for a while.
