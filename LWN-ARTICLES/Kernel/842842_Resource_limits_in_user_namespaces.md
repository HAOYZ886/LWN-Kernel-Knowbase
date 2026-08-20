---
title: Resource limits in user namespaces
url: https://lwn.net/Articles/842842/
date: "January 18, 2021"
category: "Namespaces-User namespaces; Resource limits"
author: "By Jonathan Corbet January 18, 2021"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
January 18, 2021

[User namespaces](<https://lwn.net/Articles/532593/>) provide a number of interesting challenges for the kernel. They give a user the illusion of owning the system, but must still operate within the restrictions that apply outside of the namespace. [Resource limits](<https://man7.org/linux/man-pages/man3/vlimit.3.html>) represent one type of restriction that, it seems, is proving too restrictive for some users. [This patch set](<https://lwn.net/ml/linux-kernel/cover.1610722473.git.gladkov.alexey@gmail.com/>) from Alexey Gladkov attempts to address the problem by way of a not-entirely-obvious approach. 

Consider the following use case, as stated in the patch series. Some user wants to run a service that is known not to fork within a container. As a way of constraining that service, the user sets the resource limit for the number of processes to one, explicitly preventing the process from forking. That limit is global, though, so if this user tries to run two containers with that service, the second one will exceed the limit and fail to start. As a result, our user becomes depressed and considers a career change to goat farming. 

Clearly, what is needed is a way to make at least some resource limits apply on per-container basis; then each container could run its service with the process limit set to one and everybody will be happy (except perhaps the goats). One could readily imagine a couple of ways to do this: 

  * Turn the resource limits that apply globally (many are per-process now) into limits that can also be set within a user namespace. The global limit would still apply, but lower limits could be set within a namespace to get the desired effect. 
  * Create a new control-group controller to manage resource limits in a hierarchical manner. This kind of control, after all, is just what control groups were created for. 

Gladkov's patch set, though, takes neither of those approaches. Instead, this patch set moves a number of global resource-usage counters (processes, pending signals, pages locked in memory, bytes in message queues) into the [`ucounts` structure](<https://elixir.bootlin.com/linux/v5.11-rc3/source/include/linux/user_namespace.h#L91>) associated with user namespaces. That makes the _tracking_ of the use of these resources specific to each namespace. 

User namespaces are arranged hierarchically up to the "initial namespace" at the root, and there is a `ucounts` structure allocated for each. The resource-usage counts are managed all the way up the hierarchy. So, if a process creates a new process within a user namespace, the process count in that namespace will be incremented, but so will the counts in any higher-level namespaces. The resource limit (which is still global) is checked at every level in the hierarchy; exceeding the limit at any level is cause to block an operation. 

If one is slow and undercaffeinated like your editor, one might wonder how this is supposed to work. Yes, each user namespace will now have its own count for resources like processes. If the global limit is set to one, each user namespace can contain one process without exceeding the limit at that level. But the counts propagate upward; if both namespaces have a common parent, then the limit will be exceeded at that level and our user is left no happier than before. 

A look at [the test code](<https://lwn.net/ml/linux-kernel/3102e17576584b3bfa5854f457073a42574dec8c.1610722474.git.gladkov.alexey@gmail.com/>) provided with the patch set gives an answer. In the test program, the "server" processes are created by root before changing user and group IDs and moving into a separate user namespace. The parent namespace thus belongs to root and is not subject to any resource limits set after the user-ID change. It all works as long as one's use case matches this pattern. 

Still, one might wonder why the other approaches weren't taken. Having the limits propagate downward (rather than counts propagating upward) would seem to address this problem as well in a more flexible way that doesn't require root privileges. In fact, Linus Torvalds [asked why this approach wasn't taken](<https://lwn.net/ml/linux-kernel/CAHk-=wgXZmRu762bjSeK80+T_LTo+UP9y5rP-uvym1vquSxmBw@mail.gmail.com/>) in response to a previous version of the patch set. Eric Biederman [answered](<https://lwn.net/ml/linux-kernel/87a6tfp6sw.fsf@x220.int.ebiederm.org/>) that the limit approach ""needs to work as well"", but then reiterated the use case without really clarifying why the count-based approach is needed. 

Using control groups for this purpose was discussed back in 2015. At that time, control-group maintainer Tejun Heo [rejected](<https://lore.kernel.org/lkml/20150227170640.GK3964@htj.duckdns.org/>) the idea, calling it ""pretty silly"". He continued: 

In general, I'm pretty strongly against adding controllers for things which aren't fundamental resources in the system. What's next? Open files? Pipe buffer? Number of flocks? Number of session leaders or program groups? 

If you want to prevent a certain class of jobs from exhausting a given resource, protecting that resource is the obvious thing to do. 

That particular conversation went fairly badly downhill from there, but this specific outcome has stood over time: control-group controllers are not used for control of resource limits within containers. 

For users who are facing this problem now, the only apparent solution is Gladkov's patch set. Whether these patches are merged will, however, depend on whether the rest of the kernel community thinks that this approach is the correct one. That conversation has not yet happened, and may depend on a clearer description of the semantics of this change (and its motivation) being posted first. Resource limits within containers is a problem that has remained unsolved for years; it may take longer yet to get to the real solution. 

**Update** : as explained in the comments, resource limits are already per-process, so nothing has to be done on that side to make them adjustable on a per-container basis. The _counts_ used to enforce those limits, though, are global, causing the sort of interference described above. So the proposed solution — making the counts local while still aggregating them upward — appears to make sense.
