---
title: Initiating writeback earlier
url: https://lwn.net/Articles/1078767/
date: "June 26, 2026"
category: "Block layer-Writeback"
author: "By Jake Edge June 26, 2026 LSFMM+BPF"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
June 26, 2026

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2026/>)

Writeback is the process of ensuring that dirty pages or folios in the page cache are flushed to the disk, so that changes to those files are made persistent. In a filesystem-track session at the 2026 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>), Jeff Layton wanted to discuss whether the writeback operation should be initiated earlier than it is today. The consensus seemed to be that it should be done earlier, but the path toward making that happen was less clear. 

#### Background

In the last few months, Layton began, some new [`ioctl()`](<https://man7.org/linux/man-pages/man2/ioctl.2.html>) commands were [added to the debugfs interface for the NFS server](<https://lwn.net/ml/all/20250610205737.63343-1-snitzer@kernel.org/>) (NFSD) that provided the ability to perform "dontcache" I/O (related to `RWF_DONTCACHE`, which was [`RWF_UNCACHED` originally](<https://lwn.net/Articles/998783/>)) and direct I/O (`O_DIRECT`). The controls were added in debugfs to give developers a chance to try various things before settling on a final interface. Mike Snitzer did most of the work on that, Layton said. The [debugfs interface that was eventually added](<https://lore.kernel.org/all/20250906212511.9139-1-cel@kernel.org/>) was somewhat different than what was proposed. 

Various experiments were run with the new I/O modes to try to quantify which workloads would benefit from them. One thing became clear quickly: dontcache ""was pretty awful for writes, at least under NFSD"". There are a number of reasons for that, but the main one is that NFS clients send lots of writes in parallel, which end up being written by multiple threads in the server; that did not fit well with that implementation of dontcache for NFSD. 

He has some [patches in flight](<https://lwn.net/ml/all/20260511-dontcache-v7-0-2848ddce8090@kernel.org/>) that fix the performance problem for `RWF_DONTCACHE` users; he thanked Jan Kara, Christoph Hellwig, and Ritesh Harjani ""who helped me beat that into shape"". Since the summit, those patches have been merged for 7.2. One of the characteristics of dontcache is that it immediately submits writes to the device, which was one of the problems encountered by NFSD; all of the threads woke up and submitted I/O, which caused them to queue waiting to be able to submit. The fix puts the data into the page cache until writeback completes, and moves the writeback operation to the flusher threads; that mode now performs better than direct I/O in many cases, Layton said. 

[ ![\[Jeff Layton\]](https://static.lwn.net/images/2026/lsfmb-layton-sm.png) ](<https://lwn.net/Articles/1079158/>)

That work started him thinking about doing writeback earlier in general. The number of dirty pages that accumulate is much lower after his fixes, which led him to a question: ""are we waiting too long to kick off writes in regular buffered I/O?"" Writeback is initiated for buffered I/O when the memory-management subsystem decides that the number of dirty pages is too high; it is also initiated periodically. But ""modern memory sizes are huge"", so the default dirty limits used are out of date; he thought they had not been updated for 20 years or more. That is really a separate problem and one that administrators ""can tune around"", but it may make sense to look at changing the defaults. 

He wondered if relying on memory management to decide when to do writeback was really the right approach. Maybe the filesystem layer should be more proactive; some calculation based on the backing store and number of dirty pages bound for it could be used to determine when to initiate writeback. The Linux philosophy is that free memory is wasted memory, which is why the page cache, for example, exists, but it is also true that unused disk bandwidth is wasted. 

He suggested having a per-superblock dirty limit that was a lot lower than what memory management would normally use. Starting writeback earlier ""would help keep us out of the weeds as far as dirty pages go"". Outside of the session, Kara had pointed out a potential problem: there are workloads that create lots of temporary files and then delete them. Those files are typically deleted before they get written today, so starting writeback earlier might cause them to be written unnecessarily. A lot of that kind of work is done on tmpfs these days, Layton thinks, so maybe the problem is less critical. Kara had suggested skipping writeback for inodes that have no directory entries (dentries), which might be a way to avoid much of the problem as well. 

#### Discussion

Layton said that he was just putting those ideas out for discussion; he did not have any fixed views on where to go with them. ""Any thoughts?"" 

Ted Ts'o said that he had some ideas, but that they might be somewhat contradictory. He wondered how important it was for the kernel to save battery power by not spinning up disks frequently; he noted that there are few laptops with those kinds of disks anymore so maybe [laptop mode](<https://www.kernel.org/doc/Documentation/laptops/laptop-mode.txt>) is a thing of the past. (It turns out that laptop mode was [removed for the 7.0 kernel](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=64dd89ae01f2>).) 

There are also some kinds of files ""where a certain amount of delay is good"" because it allows the application to finish writing the file, which helps if the filesystem is doing delayed allocation. So maybe files that are open and still being written to should be deferred for writeback for a time. There are lots of policy questions that would need to be resolved for something like that, however. 

Another idea, Ts'o said, might be to try to write at 50% of the backing-device bandwidth, if that can be calculated, to try to ensure that there is headroom so that [`fsync()`](<https://man7.org/linux/man-pages/man2/fsync.2.html>) latency is reasonably low. One could imagine an auto-tuning process that reserved more headroom for `fsync()`-heavy workloads. As he noted, some of his ideas ""are in tension with one another"", so he would prefer that the heuristics used are automatic. Relying on humans to change various parameters is not a good strategy: it ""never happens"". Layton noted that reserving too much backing-device bandwidth for writes might leave too little for reads, so that is another piece of the puzzle. 

Boris Burkov said that he had encountered a problem on a huge-memory system that accumulated lots of dirty pages; ""it piled up writeback until it was enormous and the resulting writeback was bad"". On March 2, he tried a change that set lower fixed dirty limits, but had to back it all out the next day ""because it broke everything"", he said to laughter. He was not suggesting that changing the default was a bad idea, because the default is also bad, ""but you're going to break _something_" ". 

Kara asked what specific problem was encountered with the lower limits. Burkov said that it was a system which downloaded and installed packages; its performance regressed after the limits were changed. Kara suggested this might be the temporary-file problem he had raised with Layton earlier; maybe the installation process was using lots of temporary files that would not actually be written with higher limits. 

Chuck Lever raised ""a converse problem"" related to NFSD. There are situations where the NFS clients are sending so many writes that NFSD simply cannot keep up ""and there doesn't seem to be an effective pushback from the server to the clients"". There are some ways to slow things down with NFSv4 or by adjusting the TCP window, but he thinks there needs to be a way for the virtual filesystem (VFS) layer to slow down writers. Neil Brown had told Lever that there already was a mechanism to do that, but that maybe it was not working correctly. ""This room is full of experts who can tell me I'm wrong, go ahead"", Lever said with a grin. 

Kara said that he ""was kind of right"" because the feedback mechanism is geared toward local filesystems where the resources available for writes are bounded by the producer sharing the system with the writeback mechanism. For NFS, though, the number of writes that clients can send is only limited by memory. There are mechanisms to throttle writers when too much data is being written, but it is apparently broken for NFS. ""Back-pressure from the server does not propagate properly"" to the client. A fast-moving conversation between Kara, Layton, and Lever seemed to conclude that there are ways to provide the back-pressure. 

Lever asked if there was any way to observe the back-pressure and resulting slowdown. Kara said that there wasn't but that it could be built; he had some suggestions on what kinds of information could be exported to help with that. 

Ts'o said that there is decades of research on ways to predict the lifetime of files based on their name and other attributes, which might be useful for determining files that should have their writeback delayed. That would be a heuristic that either requires a decision from a system administrator, ""which will never happen"", or some kind of automatic determination possibly based on some of the research. It will be important to have some observability into that process as well, he said, ""because debugging this is going to require us understanding which of our mysterious heuristics triggered—correctly or not so correctly"". 

Christian Brauner was concerned that any kind of automatic heuristic configuration in the kernel ""that fits all kinds of workloads that exist, is just doomed to fail"". He pointed to various control-group parameters that allowed the kernel ""to make this user space's problem"" and not have to come up with something that will cover everyone's needs. He thinks the mechanisms used by control groups and systemd provide a good model for how various problems like writeback limits should be handled. On the other hand, ""we have [`fadvise()`](<https://man7.org/linux/man-pages/man2/posix_fadvise.2.html>), though no one uses it"", Ts'o said to knowing laughter. 

Lever asked Layton for more information about why the original dontcache approach for NFSD did not perform well. The basic problem is that immediately performing the write from the thread that receives the client request blocks the thread until the write completes, Layton said. By moving that work to the flusher thread, the server thread can continue to handle requests while the writeback is happening in the flusher. 

Lever also wondered how the page-cache entry was dropped once the writeback is completed, since that is what `RWF_DONTCACHE` is meant to do. There is a folio flag, `PG_dropbehind`, that tracks whether to keep the folio in the cache after its writeback completes, Layton said. The only real differences between `RWF_DONTCACHE` writes and buffered writes are the setting of that bit and initiating writeback as soon as possible for `RWF_DONTCACHE`. 

After another minute or two of discussion on some other quick topics, the session ran out of time and concluded.
