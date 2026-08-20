---
title: The unified control group hierarchy in 3.16
url: https://lwn.net/Articles/601840/
date: "June 11, 2014"
category: Control groups
author: "By Jonathan Corbet June 11, 2014"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
June 11, 2014

The idea of reworking the kernel's control group implementation is not exactly new; see [this article from early 2012](<https://lwn.net/Articles/484251/>), for example. However, that talk has not yet translated into much in the way of user-visible changes to the kernel. That situation will change in the 3.16 release, which will include the new unified control group hierarchy code. This article will be an overview of how the unified hierarchy will work at the user level. 

At its core, the control group subsystem is simply a way of organizing processes into hierarchies; controllers can then be applied to the hierarchies to enforce policies on the processes contained therein. From the beginning, control groups have allowed the creation of multiple hierarchies, each of which can contain a different mix of processes. So one could, for example, create one hierarchy and attach the CPU scheduler controller to it. Another hierarchy could be created for the memory controller; it could contain the same processes, but with a different organization. That would allow memory usage policy to be applied to different groupings of the same processes. 

This flexibility has a certain appeal, but it has its costs. It can be expensive for the kernel to keep track of all the controllers that apply to a given process. Controllers also cannot effectively cooperate with each other, since they may be operating on entirely different hierarchies. In some cases (memory and block I/O bandwidth control, for example), better cooperation is needed to effectively control resource use. And, in the end, there has been little real-world use of this feature. So the plan has long been to get rid of the multiple-hierarchy feature, though it has always been known that this change would take a long time to effect fully. 

Work on the unified control group hierarchy has been underway for some time, with much of the preparatory work being merged into the 3.14 and 3.15 kernels. In 3.16, this feature will be available, but only to users who ask for it explicitly. To use the unified hierarchy, the new control group virtual filesystem should be mounted with a command like: 

```
mount -t cgroup -o __DEVEL__sane_behavior cgroup <mount-point>
```

Obviously, the `__DEVEL__sane_behavior` option is not intended to be a permanent fixture. It may still be some time, though, before the unified hierarchy becomes available as a default feature. 

It is worth noting that the older, multiple-hierarchy mode continues to work even if the unified hierarchy mode is used; it will be kept around for as long as it seems to be needed. The unified hierarchy can be instantiated alongside older hierarchies, but controllers cannot be shared between the unified hierarchy and any others. The care that has been taken in this area should allow users to experiment with the unified mode while avoiding changes that would break existing systems. 

In current kernels, controllers are attached to control groups by specifying options to the `mount` command that creates the hierarchy. In the unified hierarchy world, instead, all controllers are ![\[Control group hierarchy\]](https://static.lwn.net/images/2014/cgroup-unified.png) attached to the root of the hierarchy. (Strictly speaking that's not quite true; controllers attached to old-style hierarchies will not be available in the unified hierarchy, but that's a detail that can be ignored for now). Controllers can be enabled for specific subtrees of the hierarchy, subject to a small set of rules. For the purposes of illustrating these rules, imagine a control group hierarchy like the one shown on the right; groups A and B live directly under the root control group, while C and D are children of B. 

Each control group in the hierarchy has (in its associated control directory) a file called `cgroup.controllers` that lists the controllers that can be enabled for children of that group. Another file, `cgroup.subtree_control`, lists the controllers that are actually enabled; writing to that file can turn controllers on or off. It is worth repeating that these files manage the controllers attached to the _children_ of the group; in the unified hierarchy, a control group is thought of as delegating its resources to subgroups for management. There are some interesting implications resulting from this design. 

One of those is that a control group must apply a controller to all of its children or none. If the memory controller is enabled in B's `cgroup.subtree_control` file, it will apply to both C and D; there is no way (from B's point of view) to apply the controller to only one of those subgroups. Further, a controller can only be enabled in a specific control group if it is enabled in that group's parent; a controller cannot be enabled in group C unless it is already enabled in group B. That suggests that all controllers that are actually meant to be used must be enabled in the root control group, at which point they will apply to the entire hierarchy. It _is_ , however, possible to disable a controller at a lower level. So, if the CPU controller is enabled in the root, it can be disabled in group A, exempting all of A's descendant groups from CPU control. 

Another new rule is that the `cgroup.subtree_control` file can only be used to change the set of active controllers if the associated group contains no processes. So, for example, if group B has controllers enabled in its `cgroup.subtree_control` file, it cannot contain any processes; those processes must all be placed into group C or D. This rule prevents situations where processes in the parent control group are competing with those in the child groups — situations that current controllers handle inconsistently and, often, badly. The one exception to the "no processes" rule is the root control group. 

One other control file found in the unified hierarchy is called `cgroup.populated`; reading it will return a nonzero value if there are any processes in the group (or its descendants). By using `poll()` on this file, a process can be notified if a control group becomes completely empty; the process would presumably respond by cleaning up and removing the group. Current kernels, instead, create a helper process to provide the notification; this technique has been frowned on for years. 

The unified hierarchy will allow a privileged process to delegate access to control group functionality by changing the owner of the associated control files. But this delegation only works to an extent: a unprivileged process with access to the control files can create child control groups and move processes between groups, but it cannot change any controller settings. This policy is there partly to keep unprivileged processes from disrupting the system, but the intent is also to restrict access to the more advanced control knobs. These knobs are currently deemed to expose too much information about the kernel's internals, so there is a desire to avoid having applications depend on them. 

All of this work has been extensively discussed for years, with most of the major users of control groups having had their say. So it should be suitable for most of the known uses today, but that is no substitute for actually seeing things work. The 3.16 kernel will provide an opportunity for interested users to try out the new mode and find out which problems remain; actual migration by users to the new scheme cannot be expected to happen for a few more development cycles at the earliest, though. But, at some point, the control group rework will cease being something that's mostly talked about and become just another big job that eventually got done.
