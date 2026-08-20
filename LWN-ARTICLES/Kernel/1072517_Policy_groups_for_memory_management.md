---
title: Policy groups for memory management
url: https://lwn.net/Articles/1072517/
date: "May 14, 2026"
category: "Memory management-Control groups"
author: "By Jonathan Corbet May 14, 2026 LSFMM+BPF"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
May 14, 2026

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2026/>)

The kernel's [control-group subsystem](<https://docs.kernel.org/admin-guide/cgroup-v2.html>) works well for resource management, Chris Li said at the beginning of his memory-management-track session at the 2026 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>). Control groups work less well for other use cases, though. He was there to present his proposed enhancement, called "policy groups", that would address some of the shortcomings that he has encountered. A consensus on how this feature should look still seems distant, though. 

[![\[Chris Li\]](https://static.lwn.net/images/conf/2026/lsfmm/ChrisLi-sm.png)](<https://lwn.net/Articles/1072520/>) Resource management is designed deeply into control groups; this focus drives a core assumption that the resources granted to a parent group must be greater than or equal to the resources given to any of its child groups. Control groups are also organized into a unified hierarchy; that was a key requirement of [the control-group redesign effort](<https://lwn.net/Articles/484251/>) over a decade ago. But this design has limitations; it does not fit cases that do not conform to its resource-management model, the unified hierarchy does not work for all use cases, and control groups are not an effective tool for policies that are not tied to processes. 

As an example of a policy that doesn't fit the resource model, he said, consider service-level objectives. A child group might be given a service level that is either faster or slower than its parent. An area of particular interest to Li is regulating access to swap devices of different speeds; the difficulty in adapting control groups to this model is impeding the upstreaming of the [swap-tiers work](<https://lwn.net/ml/all/20260126065242.1221862-1-youngjun.park@lge.com/>). A case that doesn't fit the unified hierarchy would be the Android distinction between foreground and background tasks; applications can perform some of that organization internally, but what they come up with may not fit the system's view of the process hierarchy. Non-process cases include control over filesystem allocation and network-control policies. 

The proposed policy groups, which would be attached to control groups in an unspecified way, would address these limitations. Policy groups would be focused on managing policies rather than resources and would not be forced into the same hierarchical model. There have been other attempts at this sort of control, he said, including network namespaces, NUMA memory policies, and the use of [`prctl()`](<https://man7.org/linux/man-pages/man2/prctl.2.html>) to control behaviors like [kernel samepage merging](<https://lwn.net/Articles/953141/>). Policy groups would bring a more formalized structure to this kind of feature. 

Liam Howlett said that policy-related features typically use `prctl()`, and asked whether that is really a good fit for this task. There are, he suggested, a lot of features stuffed into `prctl()` that should perhaps be implemented differently. Suren Baghdasaryan asked why policy groups would be associated with control groups if the policies to be enforced are not hierarchical in nature; Li answered that there is still a need to attach policies to the process hierarchy. 

Lennart Poettering said that the control-group redesign moved that subsystem away from independent hierarchies, which was a good thing; it would be better to avoid bringing that concept back. In more recent kernels, it is possible to attach extended attributes to control groups; these, he suggested, could be used to attach policies. The [BPF Linux security module](<https://docs.kernel.org/bpf/prog_lsm.html>) uses extended attributes attached to control groups in this way. Extended attributes, he said, might well be a good fit for policy groups as well. Li answered that this approach might work for some cases, but not for those that are not inherently tied to processes. 

Roman Gushchin said that policy groups probably should not be attached to control groups at all. Another participant said that grouping all these policies under a single framework might be a mistake. It could be better, he said, to attach some policies to a filesystem, and others to a control group, for example. While an overall policy framework might be useful, nobody has ever figured out a generally applicable solution. 

Li asked whether the right approach might be to create a new policy-group virtual filesystem; it might present a flat view rather than implementing a hierarchy. Poettering answered that he is not looking forward to dealing with yet another control interface from the kernel. Li asked how Poettering would suggest setting a process's service level for swap; Poettering repeated the extended-attributes idea. 

The discussion lost focus as time ran out; a suggestion to add a new namespace type did not get a lot of support. It was seemingly agreed that it would be better to not add a new control structure if possible. Using BPF was suggested, but there are systems (especially in the embedded area) that do not support BPF, so Li said he would prefer to avoid that approach. The session closed with Li saying that he would look more closely at ways of attaching policies directly to processes.
