---
title: "Dynamically allocated pseudo-filesystems"
url: https://lwn.net/Articles/895111/
date: "May 16, 2022"
category: "Filesystems-Pseudo"
author: "By Jake Edge May 16, 2022 LSFMM"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jake Edge**  
May 16, 2022

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2022/>)

It is perhaps unusual to have a kernel tracing developer leading a filesystem session, Steven Rostedt said, at the beginning of such a session at the [2022 Linux Storage, Filesystem, Memory-management and BPF Summit](<https://events.linuxfoundation.org/lsfmm/>) (LSFMM). But he was doing so to try to find a good way to dynamically allocate kernel data structures for some of the pseudo-filesystems, such as sysfs, debugfs, and tracefs, in the kernel. Avoiding static allocations would save memory, especially on systems that are not actually using any of the files in those filesystems. 

#### Problem

[ ![\[Steven Rostedt\]](https://static.lwn.net/images/2022/lsfmm-rostedt-sm.png) ](<https://lwn.net/Articles/895267/>)

He presented some statistics on the number of files and directories on one of his systems in `/sys`, `/proc`, `/sys/kernel/tracing` (the usual mount point for tracefs), and `/sys/kernel/debug` (debugfs). In all, he found 29,384 directories and 290,807 files. That's a lot of files, but, he asked, why should he care about that? To answer that, he noted that at one point, he had suggested that Alexei Starovoitov use tracing instances, which add another set of ring buffers for trace events and add a bunch of control files in tracefs. But Starovoitov tried that and complained that new instances used too much memory. The ring buffers are fairly modest in size, a bit over a megabyte per CPU, so Rostedt dug in a bit deeper. It turns out that whenever another instance gets added to tracefs, it adds around 18,000 files. Adding up the in-memory size of the inodes and directory entries (dentries) shows that 14MB is consumed for each tracing instance that gets added. 

Looking beyond that, `/sys` consumes 42MB and `/proc` uses a whopping 202MB for these in-memory inodes and dentries, he said. But David Howells pointed out that `/proc` does not keep dentries and inodes around. Rostedt said that if he can use the same technique as procfs, "my talk is over". Ted Ts'o cautioned that it was a procfs-specific hack that had never been generalized, though Howells thought that perhaps it could be. 

On the other hand, Chris Mason looked at a Meta production server to see what its `/proc` looked like; a `find` from the root took multiple minutes, and pegged the CPU at 100%, to find that there were 31 million files in it. He suggested that the procfs-specific hack "might not be the right hack" to use. 

Christian Brauner said that since tracefs is its own filesystem, the procfs technique could simply be used there. But Rostedt was adamant that he did not want a hack just to fix the problem for tracefs; he wanted to find a proper solution that could be generalized for others to use. There should be a generic way for any pseudo-filesystem to opt into a just-in-time mode, where the inodes and dentries are allocated when the files and directories are accessed. 

#### eventfs

Rostedt noted that Ajay Kaher gave a [presentation](<https://lpc.events/event/11/contributions/1085/>) at the [2021 Linux Plumbers Conference](<https://lpc.events/event/11/page/103-lpc-2021-overview>) (LPC) on eventfs, which dynamically allocates the dentries and inodes for all of the tracing events that appear in tracefs. It is a kind of sub-filesystem for tracefs to handle the event files dynamically so that new instances do not consume so much memory. It only does the dynamic allocation for the events, and not for the other control files that appear in tracefs, Rostedt said. He did some testing with and without eventfs and found that it made a huge difference. Creating a new instance without eventfs used around 11MB extra, while doing that with eventfs only used about 1MB. At LPC, some attendees said that the feature is something that should be added as an option for all pseudo-filesystems, which is what brought Rostedt to LSFMM. He wanted to get a sense for the best way to accomplish this goal and to figure out what the internal API would look like. 

In particular, since the event dentries and inodes are only present while they are being used, at least in eventfs, he is concerned that the API needs to have a way to keep them in memory while a trace involving them is running. The worry is that memory pressure could cause eventfs to be unable to create the file to disable the event. David Howells suggested that an emergency pool could be used to handle that particular problem. 

Brauner asked which API was used for tracefs; did it use the sysfs API, for example? Rostedt said that tracefs has its own API and is completely separate from any of the other pseudo-filesystems. Tracefs came about because people wanted tracing information available on production systems but did not want to build debugfs into them. So, at Greg Kroah-Hartman's suggestion, Rostedt started with the debugfs code and turned it into tracefs. 

Since tracefs has its own API, and does not rely on sysfs or kernfs, for example, that gives it more leeway to define an API for the just-in-time feature without having to convert the others, Brauner said. He thinks it will be difficult to come up with something that could be shared between tracefs and procfs, however, because procfs is so special. 

Rostedt said that perhaps tracefs "could be the guinea pig" for the feature, then other filesystems could convert over in time if that was seen as useful. He too wonders if procfs is too special to fit in, however. Mason's concern about procfs being slow because it creates its entries on the fly may also mean that other filesystems will not want the feature. Howells said with a chuckle that if Rostedt wanted to thoroughly test the feature, "putting it in procfs would be one good way to do that". 

#### Approach

Currently eventfs covers just a portion of the control files in tracefs; Rostedt would like to handle all of the tracefs files that way. But the feedback he has gotten from virtual filesystem (VFS) layer developers is that this should not be done solely for tracefs, so he was wondering what the right approach would be. 

Amir Goldstein asked if Rostedt had talked with Kroah-Hartman to see if he would be interested in this feature for debugfs. It would seem that debugfs might also benefit from it. Rostedt said he had not asked Kroah-Hartman about that. But Brauner said that debugfs and sysfs have an ingrained idea that it is the responsibility of the creator of the directories and files to clean them up, which is different from the centralization in eventfs (or something along those lines); it might be difficult to rework those other filesystems to use a different model. 

Rostedt is also concerned about race conditions and lock-ordering problems, based on his review of the eventfs code. Howells said those kinds of problems "have all been pretty well sorted in procfs". Processes come and go, as do their entries in procfs, even if they are being used. Procfs has its own structure that describes just the pieces it needs, he said, and it creates dentries and inodes on demand. It already deals with the problem of the process directory going away when the process does, though files in that subtree may still be open. 

Rostedt wondered whether he should continue working on eventfs with Kaher or if they should drop that and try to make it work for all of tracefs. Eventfs might make a good test case for where the problem areas are. Brauner asked if there were other users who wanted this functionality, which might help guide which way to go. Howells reiterated the idea that procfs might provide the best model to look at since it already handles many of the same kinds of problems. 

Overall, Rostedt said that he was not hearing anyone argue that he should not continue working on the idea. In addition, he said that he now has some good ideas of what code to look at as well as names of people to ask questions of. Patches are presumably forthcoming once he and Kaher determine the path they want to pursue.
