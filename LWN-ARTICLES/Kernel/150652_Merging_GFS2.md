---
title: Merging GFS2
url: https://lwn.net/Articles/150652/
date: "September 7, 2005"
category: "Clusters-Filesystems; Distributed lock management; Filesystems-Cluster; GFS"
author: 
---

Andrew Morton has stated that the OCFS2 cluster filesystem is likely to be merged for 2.6.14. OCFS2 is not the only such filesystem under development, however, and the developers behind the GFS2 filesystem are [wondering](<https://lwn.net/Articles/150653/>) when it, too, might be merged - into -mm, at least. Much work has been done on GFS to address [concerns](<http://lwn.net/Articles/147043/>) which have been raised previously; the developers think that it is getting close to ready for wider exposure. The resulting discussion raised a couple of interesting questions about the kernel development process. 

The first one was [asked](<https://lwn.net/Articles/150654/>) by Andrew Morton: "why?". Given that OCFS2 is going in, does the kernel really need another clustered filesystem? What, in particular, does GFS bring that OCFS2 lacks? The answers took two forms: (1) Linux has traditionally hosted a large variety of filesystems, and (2) since cluster filesystems are relatively new, users should be able to try both and see which one works better for them. David Teigland also posted [a list of GFS features](<https://lwn.net/Articles/150655/>). 

GFS will probably win this argument; there is a clear user community, and filesystems tend not to have any impact on the rest of the kernel. But, still, some developers are starting to wonder; consider, for example, [this message](<https://lwn.net/Articles/150657/>) from Suparna Bhattacharya: 

And herein lies the issue where I tend to agree with Andrew on -- its really nice to have multiple filesystems innovating freely in their niches and eventually proving themselves in practice, without being bogged down by legacy etc. But at the same time, is there enough thought and discussion about where the fragmentation/diversification is really warranted, vs improving what is already there, or say incorporating the best of one into another, maybe over a period of time? 

The other issue which came up was the creation of a user-space API for the distributed lock manager (DLM) used by GFS. If nothing else, the two cluster filesystem should have a common API so that applications can be written for either one. One option for this API might be "dlmfs", a virtual filesystem used with OCFS2. The dlmfs approach allows normal filesystem operations to be used for lock management tasks; even shell scripts can perform locking. Concerns with dlmfs include relatively slow performance and [a certain unease](<https://lwn.net/Articles/150660/>) with aspects of the interface: 

Actually I think it's rather sick. Taking O_NONBLOCK and making it a lock-manager trylock because they're kinda-sorta-similar-sounding? Spare me. O_NONBLOCK means "open this file in nonblocking mode", not "attempt to acquire a clustered filesystem lock". Not even close. 

(Andrew Morton). 

It is not clear that better alternatives exist, however. One could implement it all with a big set of `ioctl()` calls, but nobody really wants to do that. Another approach would be to create a new set of system calls specifically for lock management. Some have argued in favor of system calls, but others, such as Alan Cox, are [strongly opposed](<https://lwn.net/Articles/150663/>): 

Every so often someone decides that a deeply un-unix interface with new syscalls is a good idea. Every time history proves them totally bonkers. There are cases for new system calls but this doesn't seem one of them. 

Alan lists a number of reasons why a file descriptor-based approach makes sense for this sort of operation - they mostly come down to well-understood semantics and the fact that many things just work. 

This is clearly a discussion which could go on for some time. Daniel Phillips [points out](<https://lwn.net/Articles/150665/>) that this is not necessarily a problem. There are currently no user-space users of any DLM API beyond a few filesystem management tools, so there is no great hurry to merge any API. The cluster filesystems could go in without any user-space DLM interface at all while the developers figure out what that interface should be. And, says Daniel, perhaps there should not be one at all. Despite the perceived elegance of having a single lock manager on the system, having user space rely upon its own, user-space DLM is a workable solution which could simplify the kernel side of things.
