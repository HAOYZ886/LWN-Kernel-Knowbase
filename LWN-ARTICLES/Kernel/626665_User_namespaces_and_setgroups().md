---
title: User namespaces and setgroups()
url: https://lwn.net/Articles/626665/
date: "December 17, 2014"
category: "Namespaces-User namespaces; Negative groups; Security-Namespaces"
author: "By Jonathan Corbet December 17, 2014"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
December 17, 2014

Back in November, we [looked at a patch](<https://lwn.net/Articles/621612/>) that would allow unprivileged processes to drop groups from their credentials. After that patch was posted, it was quickly shown that, in some cases, dropping groups leads to an increase in privilege; the patch in question has not been pursued since. But it was also shown that an unprivileged user can already drop groups by making use of user namespaces. It took some time, but namespace developer Eric Biederman has put together a set of patches that, he hopes, will close that vulnerability. 

Group membership can be used to restrict privilege in a couple of ways. Access control lists can explicitly block access to a resource on the basis of membership in a particular group. But it is even simpler than that: if a file's protection bits are set for "no group access," a process belonging to that group will be blocked, even if the file is otherwise accessible by the world as a whole. In either case, the ability to drop a group can enable a process to access a resource that would have otherwise been denied to it. 

In current kernels, using `setgroups()` to change a process's group membership is a privileged operation. So unprivileged processes cannot use it to get rid of any inconvenient group memberships. But a process running within a user namespace is privileged inside that namespace, so a `setgroups()` call there will succeed. It is easy to write a little program that uses `clone()` to create a child in a user namespace and has the child call `setgroups()` to drop membership in all supplementary groups. This privilege-escalation vulnerability has become known as CVE-2014-8989. 

Eric's [fix](<https://lwn.net/Articles/626692/>) for this problem starts by disabling the use of `setgroups()` within a user namespace until a group-ID mapping has been set up for that namespace. That mapping is created by writing the file `gid_map` in the process's `/proc` directory; see [this article](<https://lwn.net/Articles/532593/>) for details on how the mapping files work. Other user- or group-ID-oriented system calls require the existence of a mapping before they will succeed; `setgroups()` now has that restriction as well. 

The biggest part of the patch adds a new control file, called `setgroups`, to the `/proc` directory for each process. Writing the string "`deny`" to that file will disable the `setgroups()` system call entirely within the namespace containing the relevant process. The `CAP_SYS_ADMIN` capability is required, so random processes cannot disable `setgroups()` in the top-level namespace; once again, a process within its own user namespace is privileged (by default) and can make this change successfully. Once `setgroups()` has been turned off, it cannot be enabled again in that namespace or any of its descendants. The `setgroups` file can only be written to _before_ the group-ID mapping has been set. 

Finally, an _unprivileged_ process can only change the group-ID mapping of a namespace if `setgroups()` has been disabled. The only thing an unprivileged process can do with the group-ID mapping is to map its own primary group ID to the same ID in the parent namespace; an unprivileged process is not able to remap its supplementary groups. So, with this set of restrictions in place, it essentially become impossible to (1) play tricks with mappings to drop groups, or (2) call `setgroups()` at all without privilege. 

Note that if a privileged process creates a user namespace, it can set up arbitrary mappings for group IDs and decline to disable `setgroups()`. That would make the dropping of groups within the namespace possible, but, since the process is already privileged, it could do that anyway. 

The end result of all this work should be the closing of the vulnerability caused by being able to drop groups within a user namespace. But it highlights one of the hazards that come with the user namespace territory: while it seems possible to contain privilege within a user namespace, there is always the possibility of surprises like this one hiding in the corners of the system. It may be some time yet before we can be truly confident that all of those surprises have been found and that the unprivileged creation of user namespaces is truly a safe thing to allow. 

Eric has [asked Linus](<https://lwn.net/Articles/626677/>) to pull these changes for the 3.19 development cycle; that pull happened just as this week's Edition was going to press. The patches have been marked for stable backporting as well, so they should eventually become available in the stable update series.
