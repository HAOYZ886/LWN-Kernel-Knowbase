---
title: Magic groups in 2.6
url: https://lwn.net/Articles/84566/
date: "May 11, 2004"
category: "Capabilities; Magic groups; Memory management-User-space memory locking"
author: 
---

The [2.6.6-mm1 tree](<http://lwn.net/Articles/84229/>) includes, among many other things, patches which add two new `/proc/sys` variables. They are: 

> `/proc/sys/vm/hugetlb_shm_group`
>     If this value is non-zero, it is interpreted as a group ID which gives access to the the "huge pages" feature of the 2.6 VM. 
> 
> `/proc/sys/vm/mlock_group`
>      This variable behaves similarly, but it controls access to the `mlock()` system call (which locks memory into physical RAM) instead. 

The current Linux kernel will not allow a process to perform either of the above actions unless that process has the `CAP_IPC_LOCK` capability; in practice, this means that the process needs to run as root. The main user of huge pages would appear to be a small program called "Oracle," which is something that many users would rather not run with root privileges. The new sysctl variables allow an administrator to give the ability to use huge pages (and `mlock()`) to a specific group; if Oracle runs within that group, it will be able to do what it needs without higher privileges. 

These patches are not universally popular; the addition of "magic groups" with special meaning inside the kernel strikes many developers as an inelegant, un-Unix-like solution to the problem. So these developers were not happy when the `hugetlb_vm_group` patch was merged for 2.6.7 shortly after appearing in the -mm tree. Rather than rush an ugly hack into the kernel (which will then have to be supported indefinitely into the future), they argue, it would have been better to come up with a proper solution. 

The problem, it seems, is that there are no better solutions on the horizon. [Says Andrew Morton](<https://lwn.net/Articles/84571/>): 

Capabilities are broken and don't work. Nobody has a clue how to provide the required services with SELinux and nobody has any code and we need the feature *now* before vendors go shipping even more ghastly stuff. 

The problems with capabilities were covered here [back in April](<https://lwn.net/Articles/79185/>), when this issue last came up. SELinux can, in principle, solve this problem, but there is the little disadvantage that nobody has been able to put together a production-ready, working distribution with SELinux yet. The distributors have been creating their own patches to enable Oracle to use the huge pages feature, and many of those are seen as being worse than the "magic groups" approach. Rather than see each distribution take the kernel in a different direction, Andrew [merged the magic groups patch](<https://lwn.net/Articles/84573/>) as the least evil alternative: 

Nasty workarounds will be shipped to end users by vendors. That's a certainty. We cannot change this now. What I wish to do is to ensure that all users receive the *same* nasty workaround. Call it damage control. 

To some, however, the control appears worse than the damage. If vendors add their own hacks, they take responsibility for maintaining those hacks, or for weaning users off of them at some future time. Pulling features out of the mainline kernel is harder. Be that as it may, for lack of a better short-term solution the "magic groups" patch is now part of 2.6.
