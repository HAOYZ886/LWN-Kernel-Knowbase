---
title: Namespaced file capabilities
url: https://lwn.net/Articles/726816/
date: "June 30, 2017"
category: "Capabilities; Namespaces-User namespaces"
author: "By Jonathan Corbet June 30, 2017"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
June 30, 2017

The kernel's file capabilities mechanism is a bit of an awkward fit with user namespaces, in that all namespaces have the same view of the capabilities associated with a given executable file. There is [a patch set under consideration](<https://lwn.net/Articles/726273/>) that adds awareness of user namespaces to file capabilities, but it has brought forth some disagreement on how such a mechanism should work. The question is, in brief: how should a set of file capabilities be picked for any given user namespace? 

The Linux [capabilities](<http://man7.org/linux/man-pages/man7/capabilities.7.html>) mechanism is meant to allow privileges to be granted to processes in a manner that is more fine-grained than the classic Unix "root can do anything" approach. So, for example, an otherwise unprivileged program that needs to be able to send a signal to an unrelated process could be given `CAP_KILL` rather than full root privileges. Capabilities have not revolutionized privilege management as had once been hoped, but they can still have their uses. 

In a typical Unix system, privileged operations are made available to ordinary users by way of setuid programs. In a system with capabilities, it is natural to want to associate capabilities with an executable program instead, once again in the hope of limiting the amount of privilege that must be granted. [File capabilities](<https://lwn.net/Articles/211883/>), added for the 2.6.24 kernel, provide that feature. 

[User namespaces](<https://lwn.net/Articles/532593/>) allow a set of processes to run as root within the namespace, while mapping the root ID (and possibly others) to normal IDs for actions (such as filesystem access) involving the rest of the system. A process running as root within a user namespace can create a setuid-root binary that will only work as intended within that namespace; it will not be usable to escalate privileges outside of the namespace. The same is not true of file capabilities, though; all user namespaces have the same view of the capabilities associated with an executable file and, since processes in a user namespace lack privilege in the root namespace, they cannot change those capabilities. 

File capabilities are implemented using extended attributes; in particular, they are stored in the `security.capability` attribute. The kernel handles the `security.*` extended-attribute namespace specially; only a privileged program (one possessing the `CAP_SYS_ADMIN` capability in particular) can change those attributes. So it is not possible for an unprivileged container running within a user namespace to add capabilities to a file; there is, in any case, no way to store extended attributes such that they are only visible within a given user namespace. 

The proposed patch set, posted by Stefan Berger, aims to change that by extending the extended-attribute syntax. This is done by decorating attributes with syntax describing the user ID (in the root namespace) associated with UID zero within a user namespace. So, for example, if a user with UID 1000 starts a user namespace, processes running as root within that namespace will access the filesystem with the original ID of 1000\. If that user adds capabilities to a file within the user namespace, those capabilities will actually be stored in an extended attribute named: 

```
security.capability@uid=1000
```

Outside the namespace, this new attribute will have no effect. Within any namespace mapped to UID 1000, though, that attribute will appear as simply `security.capability`, so the program contained within that file will run with those capabilities in its masks. 

This mechanism does not apply to extended attributes in general; it is, instead, restricted to a specific set of attributes that the kernel cares about. In the patch set, `security.capability` is obviously one of those attributes; the other is `security.selinux`, allowing for namespace-specific SELinux labels on files. The SELinux attribute was later removed, though, after SELinux maintainer Stephen Smalley [pointed out](<https://lwn.net/Articles/726829/>) that it would not work as intended. 

Casey Schaufler [objected](<https://lwn.net/Articles/726824/>) to this mechanism, noting that if two user namespaces are both running mapped to UID 1000 and sharing a directory tree, file capabilities set in one of those namespaces will be visible in the other. He argued that the user ID is the wrong key to use for file capabilities; instead, he said, there should be some sort of persistent ID associated with the user namespace itself. Serge Hallyn (who had posted [a namespaced file-capabilities patch of his own](<https://lwn.net/Articles/721396/>) that had served as inspiration for Berger's work) [disagreed](<https://lwn.net/Articles/726825/>), though, saying that the feature was working as designed. 

James Bottomley, instead, [objected](<https://lwn.net/Articles/726826/>) that this mechanism will work poorly on systems where user IDs for containers are allocated dynamically. He asked for a simple `@uid` suffix, which would be picked up in any user namespace. Hallyn [indicated openness](<https://lwn.net/Articles/726828/>) to adding that suffix as an additional feature. 

It would seem that most of the concerns about the feature itself have been headed off, so this patch set may be well on its way toward acceptance. That does, of course, leave out the biggest point of contention of all, one that was inevitable in retrospect: the proper formatting of the namespace-specific extended-attribute names. So the final form of the attribute may be something like `security.ns@uid=1000@@.capability` when the dust settles. Otherwise, though, namespaced file capabilities may be a kernel feature in the relatively near future.
