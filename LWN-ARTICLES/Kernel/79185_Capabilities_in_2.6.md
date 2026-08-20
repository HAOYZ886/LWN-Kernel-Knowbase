---
title: Capabilities in 2.6
url: https://lwn.net/Articles/79185/
date: "April 6, 2004"
category: "Capabilities; Memory management-User-space memory locking"
author: 
---

The kernel capability mechanism gives (relatively) fine-grained control over what actions any given process can perform. The various capabilities include the ability to override file permissions, send signals to other processes, bind to low-numbered ports, and many other tasks. There have been visions over the years of exporting capabilities to user space and eliminating the "all-powerful superuser" concept, but none of those visions have been implemented in any sort of widely-distributed sort of way. 

One of the capabilities is called `CAP_IPC_LOCK`; it gives a process the ability to lock a region of virtual memory into physical RAM. This capability needs to be controlled; otherwise a rogue process could lock up all of physical memory and effectively shut down the system. There are, however, legitimate reasons for giving this capability to normal users. Programs which handle encryption (such as gpg) would like to lock in some of their memory so that passphrases and clear text do not get written out to swap. Systems like Oracle need the capability to lock in their shared segments (since they do their own paging, essentially) and to be able to allocate large page "hugetlb" segments. 

To this end, Andrea Arcangeli posted [a patch](<https://lwn.net/Articles/79110/>) which allows the system administrator to disable `CAP_IPC_LOCK` checking via a sysctl variable. With those checks disabled, any non-privileged process can lock pages into memory or allocate large-page shared memory segments. Andrea asked for the patch to be incorporated into the 2.6 mainline. 

The patch inspired some thinking on how best to make certain capabilities available to users. There has been [a patch](<http://marc.theaimsgroup.com/?l=linux-kernel&m=107980096115231&w=2>) in circulation for a while which simply opens up memory locking to everybody, but which puts a resource limit on the number of pages which can be locked. The default limit is a single page, which works for gpg but which does not easily threaten the system as a whole. With a suitably adjusted limit, this patch should work for Oracle as well - but it does not address the large-page shared memory issue. 

William Lee Irwin put together [a different patch](<https://lwn.net/Articles/79204/>) which allows the administrator to turn off checks for any capability via a set of sysctl variables. It differs from Andrea's patch in its generality, but also by virtue of using the security module framework rather than direct changes to the kernel core. Some people seemed to like this patch better, though there was some nervousness about its overall security which led William to add [a strong comment](<https://lwn.net/Articles/79205/>) and [a lockdown capability](<https://lwn.net/Articles/79206/>) to the patch. 

Given that the whole idea behind capabilities was to be able to give specific capabilities to individual users, however, some developers wondered why the current system couldn't be used. To this end, Andrew Morton [looked into hacking `login`](<https://lwn.net/Articles/79207/>) to enable it to give capabilities to users. He was not impressed with what he found once he started trying to work with kernel capabilities: 

It turns out that the whole "drop capabilities and then run something" thing does not work in either 2.4 or 2.6. And hasn't done since forever. What we have in there is no more useful than suser()... 

I must say that I'm fairly disappointed that we developed and merged all that fancy security stuff but nobody ever bothered to fix up the existing simple capability code. Particularly as, apparently, the new security stuff STILL cannot solve the extremely simple Oracle-wants-CAP_IPC_LOCK requirement. 

It was [pointed out](<https://lwn.net/Articles/79208/>) that SELinux can, in fact, solve this problem. But that will be little comfort to those who are not yet ready to adopt SELinux for their production systems. 

The problem may originate from the fact that the visions of fully capability-driven systems involve assigning capabilities to all executables and having a process's capabilities tweaked every time a new program is run. That part of the system has never been merged into the mainline, partly because nobody has ever really figured out how to deal with system administration when every file has another 32 permissions bits added onto it. The end result, in any case, is that the capability subsystem has never worked quite as it should. Given that Andrew is the gatekeeper, chances are good that some sort of fix for that problem will get into the kernel before any sort of more complicated solution to the problem of giving capabilities to users.
