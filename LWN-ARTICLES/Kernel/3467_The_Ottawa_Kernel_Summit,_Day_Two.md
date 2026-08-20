---
title: "The Ottawa Kernel Summit, Day Two"
url: https://lwn.net/Articles/3467/
date: "June 25, 2002"
category: Kernel Summit
author: 
---

[![\[Kernel hackers\]](http://old.lwn.net/images/ks/group2-sm.jpg)](<http://lwn.net/Articles/3470/?format=printable>) The second day of the Linux Kernel Developers Summit was held on June 25 in Ottawa. This writeup covers the discussions from that day; see also [our Day One coverage](<https://lwn.net/Articles/3327/>). 

## Databases

IBM's Ken Rozendal gave a presentation on what large database systems need from Linux for optimal performance. For the most part, it was a familiar shopping list: large pages, large block I/O operations, asynchronous I/O, direct I/O, multiqueue scheduler, reduced lock contention, etc. Much of that list has already found its way into the 2.5 development series, so the database people should be reasonably happy. Now it's mostly a matter of waiting for the 2.6 stable release. 

One interesting point Ken raised had to do with NUMA systems. According to Ken, most large (8-way or greater) SMP systems are really NUMA systems internally - it's just that the vendors do not present them that way. It is hard to make a multi-way system without making memory (and other resources) be "closer" to some processors than others. So various techniques for improving performance on NUMA systems (process CPU affinity, local memory allocation, text replication) will be increasingly important, even on systems which are not advertised as being NUMA. 

Linus pointed out that even desktop PCs are starting to look more NUMA-like as a result of processors with hyperthreading. 

One desired feature still does not exist in Linux: I/O completion ports. Completion ports allow a process to wait on a (potentially very large) number of I/O events, and provide a straightforward indication of which event actually occurred. Applications like the Domino server, which can be waiting on tens of thousands of network sockets, would benefit from this capability. 

As always, there remains work to do. A comparison of this presentation with the database wishlist from [last year's kernel summit](<http://old.lwn.net/2001/features/KernelSummit/>) shows, however, that a great deal of progress has been made. 

## HP's kernel wishlist

Bdale Garbee, the current Debian Project Leader, presented HP's wishlist for the Linux kernel. It was relatively short and straightforward; either HP is mostly happy with the state of the kernel, or the company hasn't gotten around to asking for the big ticket items yet. 

For example, many of HP's requests had to do with the SCSI layer, but they didn't ask for the massive rewrite that most kernel hackers seem to think is needed. HP would like to see support for the SCSI 3 REPORT LUN operation, very large numbers of SCSI logical units, better error handling, and failover support. 

Like many vendors, HP would like better support for very large systems. In particular, the company sells systems with more than 256 PCI busses installed, and would like that to work well (the support is mostly there now). Better support for discontiguous memory is also on HP's list; some of HP's chipsets impose specific memory layout requirements. 

HP, too, is getting into the "carrier grade Linux" area; it seems there are real customers wanting to spend real money on that stuff. 

Improved hotplug support made the list. HP would like to see PCMCIA devices treated like other hotpluggable devices; that would make iPAQ support easier. Better IDE hotplug support also made the list. The iPAQ folks would also like union filesystem support. 

Finally, a couple of customer support issues made the list. Customers really don't like it when their devices change names, so the old standby - table device naming - was mentioned. Bdale also mentioned support of binary modules, which is "pure hell." As Debian project leader he's not much fond of binary modules, but HP has to deal with them. Some discussion of how to better support vendors and external modules ensued, but the problem is thorny and no real solutions emerged. 

In the last part of the talk, Bdale whipped out his Debian T-shirt and raised the question: does Debian violate its Social Contract by shipping the Linux kernel? The issue, of course, is the inclusion of proprietary firmware in a number of device drivers. Kernel developers, as a whole, seem unconcerned about proprietary firmware. There can be specific problematic cases (i.e. where binary-only firmware is marked as being licensed under the GPL), but most firmware is seen as being OK. Greg Kroah-Hartman pointed out that he is willing to move firmware into a user-space task for USB devices, but that nobody has been sufficiently concerned to make a patch to do this. 

Linus stated that moving firmware into user space is "technically incredibly stupid" and "a sign of mental disorder." Nobody really challenged that claim. 

In the end, this question was pushed back to the Debian legal community, which seems to have more time to debate such issues. Just when is it permissible to have proprietary firmware for a device mixed in with GPL code? We look forward to their answer. 

## The Loadable Security Module

Chris Wright presented the Loadable Security Module patch. This patch has its roots in the previous kernel summit, where Linus asked for a standard mechanism by which enhanced security regimes could be loaded into the kernel. The LSM patch places about 150 hooks throughout the kernel; each hook, essentially, provides a security module with the opportunity to veto an action by some process. 

The LSM interface has been stable for about six months, and a number of security mechanisms have been implemented on top of it. Chris stopped short of saying that it is ready for merging into the kernel, but he did say it's at a point where it needs wider exposure and feedback. 

Nobody expressed outright opposition to the LSM patch, which is a good sign for its eventual inclusion. The questions were mostly oriented around performance penalties, code maintainability, and other approaches to security. 

In general, the performance cost of the LSM patch is small - 0-2% on lmbench runs. That cost is, of course, not counting the overhead imposed by any particular security regime - it is just the LSM framework itself. So, as a whole, LSM costs little; the big exception is with high-performance networking. Gigabyte Ethernet can suffer by as much as 20%. So there is interest in being able to disable the LSM calls for low-level networking only. (**Update** : the 20% performance cost, it seems, is an SELinux issue, and is not caused by the LSM framework itself. I'm told the real LSM penalty is closer to 5% - still significant, but not on the same scale.) 

The LSM patch is intrusive by its nature - those 150 hooks have to be spread throughout the kernel. The changes are small, however; they consist mostly of a call into the security module and a possible error exit. There was some exploration of possibly less intrusive ways of hooking in security checks: checking at the system call boundary or using `ptrace()`. But checking in this way is far less efficient, and it is also subject to race conditions that could be used to undermine the security of the system. 

The LSM team went away without a great deal of feedback beyond a need to look at the worst of the performance problems. Chances are that LSM will make an appearance sometime in 2.5. 

## Asynchronous I/O

Ben LaHaise led a session on asynchronous I/O; he was mostly looking for feedback on how he should proceed with the AIO patch to get it to where it could be merged into the mainline kernel. He got what he was after. 

Ben started with a quick status update. Some small pieces of the AIO functionality have been merged recently, including the function callbacks on wait queues. Most of the code, however, remains outside the tree: the AIO syscalls, the "work todos" functionality, kvecs, the generic file read/write code, and the driver changes. Not all of that code is ready for merging; the system calls work well, but the kvecs are controversial and the "work todos" need work still. There is a fair amount of boilerplate and duplicated code associated with AIO support; that needs to be trimmed down. 

The real question that Ben had was: to what extent should AIO support be worked into the kernel? A complete AIO implementation would affect almost all parts of the kernel and would break a lot of things. A related question was the form that AIO support should take. The current patch sets up a new set of file operations for AIO, which is handled as an operation that is distinct from regular, synchronous I/O. An alternative would be to make _all_ I/O within the kernel be asynchronous; then synchronous semantics could be implemented with an explicit wait in the relevant system calls. 

The answer from Linus seems to be to go for it: all I/O within the kernel should be asynchronous. Kernel interfaces should, in general, be asynchronous, and having two separate interfaces to do the same thing is not a good idea. Thus, Ben should go ahead and implement fully asynchronous I/O, and if that change breaks a lot of drivers for a while so be it. The drivers that matter to people will get fixed, sooner or later. 

The discussion wandered into a number of implementation details that are not necessarily of interest here. One that is worth pointing out is the matter of "kvecs." The kvec is Ben's lighter-weight answer to the "kiovec;" it is a way of representing an I/O operation at the lowest level, with pointers directly to the physical pages involved. The kernel currently has several data structures with this same basic task: kvecs, kiovecs, and the bio structure used in the block layer. It was agreed that unifying these structures would be a good idea. 

The path ahead looks fairly clear, and it should lead to a much more capable and coherent I/O subsystem. It will also be disruptive, however; expect a lot of things to break between here and the feature freeze. Of course, that is what development kernels are for. 

## SCSI

James Bottomley led a session on the status of the SCSI layer, and what needs to be done with it. Criticizing the SCSI code is a common kernel developer passtime, of course, so there was great interest in hearing what was to be done to fix it up. 

James started by posing the question: is the SCSI layer _really_ as bad as people make it out to be? His position is that, in fact, the SCSI layer is in better shape than most people think. The worst part is the error handler; it needs to be torn out and redone. Beyond that, there is a certain amount of cruft in the rest of the code (though not as much as there used to be), but as a whole it works as it should. 

Much of what is now seen as cruft was once necessary - it implemented support features that were not, at the time, available in the kernel. As the kernel matures, generic kernel code can replace things that were previously done at the SCSI level. The most recent example of this process is tagged command queueing, which is currently implemented in the low-level SCSI adapter drivers. Now that the block layer understands TCQ, this code can (eventually) be taken out of the SCSI layer. 

The real plan, though, as solidified in this session, is to eliminate most or all of the SCSI "midlayer" altogether. The midlayer sits between the kernel, the high-level SCSI drivers (i.e. the disk and tape drivers), and the low-level (adapter) drivers. Increasingly the tasks it handles are being merged into the block code, and the midlayer should shrivel away. 

Quite a bit of work will be required before this task is complete, however; there are a number of things that the SCSI midlayer does that will have to be worked into the block layer. James also does not want to rush into a major thrashup of this code; as he put it, SCSI is "Linux's key to the enterprise" and it absolutely has to work well. He said, to general applause, that SCSI changes need to be well tested before merging into the kernel, unlike the development model used in that other low-level disk subsystem. 

Various other issues were discussed. Handling of multipath devices is an open problem; it's not clear whether the multipath handling should be done at a very low level (in the adaptor driver), a very high level (in the block layer), or somewhere in between. The SCSI code should perform "lazy allocation," where it does not allocate data structures for (or even spin up) drives until it knows that they will be used. Storage-area networks can present thousands of drives, most of which will never be accessed by any given system. The current scheme of setting up for every drive was compared to the networking system creating data structures for every other system it finds on the net. 

How much work will be done in the 2.5 series remains unclear. In the end, though, the SCSI layer is likely to look a lot smaller and lighter than it does now. 

## Kernel release management

The final session of the day had to do with release management, and how to make the 2.6 stable release work better than 2.4 did. Ted Ts'o started out by stating that the usual "out of the blue" feature freezes tend to be self defeating. As soon as the freeze is decreed, Linus gets buried under a big pile of patches. Even if the kernel was relatively stable before all those patches show up, it isn't afterwards. A better way, said Ted, would be to establish a set of features for 2.6 and a well-known freeze date. That should prevent the "thundering herd of patches" problem. 

The question came up of whether there should be a "backport" version of the 2.4 kernel with some of the 2.5 features - things like the O(1) scheduler, for example. It was also suggested that perhaps the 2.6 kernel could be stabilized and released _before_ some of the remaining big changes - such as reverse mapping VM and asynchronous I/O - go in. Neither of these ideas was greated with great enthusiasm. Marcelo, the 2.4 maintainer, stated that he would rather see vendors handle backporting and packaging of 2.5 features themselves if they feel the need to do so. And a stable series without the other pending important changes did not seem interesting. 

Should perhaps the 2.7 development series open at the same time as the 2.6 release happens, or shortly thereafter? A new development kernel would certainly relieve some of the pressure that tends to push late changes into a "feature frozen" kernel. Linus worries, though, that a new development kernel would distract too many developers, and the bugs in the stable kernel simply would not get fixed. 

Then the idea came up that the maintainer of the next stable series should be named during the feature freeze stage, and should take over the stable releases from the beginning. That person would work with Linus to determine which patches should go in before the stable release. The idea is that whoever will be dealing with problems in the stable kernel would be well motivated to keep disruptive changes out. This idea was well received, even by Linus, and looks likely to happen. 

That, of course, begs the question of just who would be the next stable series maintainer. Volunteers were notably scarce in the room; some developers were seen hiding under the tables when the question was raised. The general sympathy was toward drafting either Andrew Morton or Dave Jones for the job. Neither stated that he would do it, however. 

Finally, it came down to deciding what would go into 2.6, and when the feature freeze would happen. Linus was not interested in establishing a feature list; he would rather see what comes up. He did go for a date, though: the 2.5 feature freeze is scheduled for next Halloween, October 31, 2002. That should make for interesting names: shall this be the "greased ghost" release? Or just "trick or treat"? 

There was a fair amount of optimism at the end of this session; chances seem good that the release process will go better this time around. Of course, there is still a lot of time for things to go wrong between now and then... 

[ ![\[Linus lies low\]](http://old.lwn.net/images/ks/ltflat-sm.jpg)](<https://lwn.net/Articles/3472/?format=printable>)
