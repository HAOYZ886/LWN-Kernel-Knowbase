---
title: Sticky groups in the shadows
url: https://lwn.net/Articles/855943/
date: "May 14, 2021"
category: "Namespaces-User namespaces; Negative groups"
author: "By Jonathan Corbet May 14, 2021"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
May 14, 2021

Group membership is normally used to grant access to some resource; examples might include using groups to control access to a shared directory, a printer, or the ability to use tools like `sudo`. It is possible, though, to use group membership to _deny_ access to a resource instead, and some administrators make use of that feature. But groups only work as a negative credential if the user cannot shed them at will. Occasionally, some way to escape a group has turned up, resulting in vulnerabilities on systems where they are used to block access; despite fixes in the past, it turns out that there is still a potential problem with groups and user namespaces; [this patch set](<https://lwn.net/ml/linux-kernel/20210510130011.1441834-1-gscrivan@redhat.com/>) from Giuseppe Scrivano seeks to mitigate it through the creation of "shadow" groups. 

There are two ways to prevent access to a file based on group membership. One of those is to simply set the group owner of the file to the group that is to be denied, then set the permissions to disallow group access. Members of the chosen group will be denied access to the file, even if the world permissions would otherwise allow that access. The alternative is to use [access control lists](<https://man7.org/linux/man-pages/man5/acl.5.html>) to explicitly deny access to the intended group or groups. Once again, any process in any of the designated groups will not be allowed access. 

By way of a refresher, it's worth remembering that Linux has two separate concepts of group membership. The "primary group" or "effective group ID" is the group that will be attached to new files in the absence of other constraints. This was once the only group associated with a process in Unix systems, and is set with [`setgid()`](<https://man7.org/linux/man-pages/man2/setgid.2.html>). The "supplementary" groups are a newer addition that allow a process to belong to multiple groups simultaneously; the list of supplementary groups can be changed with [`setgroups()`](<https://man7.org/linux/man-pages/man2/getgroups.2.html>). Negative access-control decisions are usually (but not necessarily) based on supplementary group membership. 

The "negative groups" access-control technique will prove porous, though, if processes are allowed to shed group membership at will. Both `setgid()` and `setgroups()` are privileged operations so, in normal circumstances, group membership is not under the control of the process involved. At least, that is true until user namespaces enter the picture. 

Back in 2014, [a problem with user namespaces and groups](<https://lwn.net/Articles/626665/>) came to light. Any user can create a user namespace and run as root within that namespace; as a result, actions that are blocked outside of the namespace (`setgroups()`, for example) become possible inside. User namespaces thus made it easy for a user to evade being hampered by membership in the wrong group; all that was needed was to create a namespace, then call `setgroups()` to drop membership of that group inside the namespace. User namespaces were designed to not confer any extra privilege outside of the namespace, but the _removal_ of a credential was not originally seen as raising privilege. 

The solution adopted at the time was to add a control file (called `setgroups`) to each process's `/proc` directory. Writing "`deny`" to that file will disable `setgroups()` to all processes within the user namespace containing the target process — and to all descendant user namespaces as well — while writing `allow` will enable `setgroups()`. This action must be performed before setting the group-ID map for the namespace, otherwise writing the group-ID map will enable `setgroups()`. This policy was chosen for ease of verification; there is exactly one place where `setgroups()` can be enabled for a namespace. If `setgroups()` is explicitly disabled, it will remain that way for the life of the namespace; there is no way to enable it again. 

This fix solved the problem, but at a cost: `setgroups()` exists for a reason, and there are legitimate workloads that would like to make use of it. Denying `setgroups()` will keep processes from escaping unwanted groups within a namespace, but it also prevents any other change of supplementary groups. 

To remedy this issue, Scrivano's patch set adds another possible value, "`shadow`", for the `setgroups` file. Writing `shadow` has the same effect as writing `allowed`, in that the `setgroups()` system call will be allowed inside the namespace, but there is a difference. When the `setgroups` file is written, the kernel will make a copy of the target process's supplementary groups at that time and store it with the namespace. Whenever `setgroups()` is called, this list of "shadow" groups will be appended to the groups provided by the caller, essentially requesting continued membership in all of those groups. 

In other words, the "shadow" mode makes the initial set of supplementary groups sticky. Suitably privileged processes within the namespace can use `setgroups()` to add new groups; they can also remove groups that were not part of the initial set. But the supplementary groups that the namespace was created with will always be there, even though the process can no longer see them — `getgroups()` will not list the groups that have not been explicitly requested with a `setgroups()` call. 

This particular patch has been around for a while; in the past, its complexity has not seemed to be justified by the benefits it brings. A recent [user request](<https://lwn.net/ml/linux-kernel/20210421172714.912119-1-snaipe@arista.com/>) for this feature has brought this work back to light, though. Whether it will clear the bar this time remains to be seen, but it seems likely that there will always be users who want to have both the ability to use negative group protections and to change group memberships within user namespaces.
