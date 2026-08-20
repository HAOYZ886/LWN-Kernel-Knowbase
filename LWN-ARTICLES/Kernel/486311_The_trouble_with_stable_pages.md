---
title: The trouble with stable pages
url: https://lwn.net/Articles/486311/
date: "March 13, 2012"
category: Stable pages
author: "By Jonathan Corbet March 13, 2012"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
March 13, 2012

Traditionally, the kernel has allowed the modification of pages in memory while those pages are in the process of being written back to persistent storage. If a process writes to a section of a file that is currently under writeback, that specific writeback operation may or may not contain all of the most recently written data. This behavior is not normally a problem; all the data will get to disk eventually, and developers (should) know that if they want to get data to disk at a specific time, they should use the `fsync()` system call to get it there. That said, there are times when modifying under-writeback pages can create problems; those problems have been addressed, but now it appears that the cure may be as bad as the disease. 

Some storage hardware can transmit and store checksums along with data; those checksums can provide assurance that the data written to (or read from) disk matches what the processor thought it was writing. If the data in a page changes after the calculation of the checksum, though, that data will appear to be corrupted when the checksum is verified later on. Volatile data can also create problems on RAID devices and with filesystems implementing advanced features like data compression. For all of these reasons, the [stable pages](<https://lwn.net/Articles/442355/>) feature was added to ext4 for the 3.0 release (some other filesystems, btrfs included, have had stable pages for some time). With this feature, pages under writeback are marked as not being writable; any process attempting to write to such a page will block until the writeback completes. It is a relatively simple change that makes system behavior more deterministic and predictable. 

That was the thought, anyway, and things do work out that way most of the time. But, occasionally, as [described](<https://lwn.net/Articles/486313/>) by Ted Ts'o, processes performing writes can find themselves blocked for lengthy periods (multiple seconds) of time. Occasional latency spikes are not the sort of deterministic behavior the developers were after; they also leave users unamused. 

In a general sense, it is not hard to imagine what may be going on after seeing this kind of problem report. The system in question is very busy, with many processes contending for the available I/O bandwidth. One process is happily minding its own business while appending to its log file. At some point, though, the final page in that log file is submitted for writeback; it then becomes unwritable. As soon as our hapless process tries to add another line to the file, it will be blocked waiting for that writeback to complete. Since the disks are contended and the I/O queues are long, that wait can go on for some time. By the time the process is allowed to proceed, it has suffered an extensive, unexpected period of latency. 

Ted's proposed solution was to only implement stable pages if the data integrity features are built into the kernel. That fix is unlikely to be merged in that form for a few reasons. Many distributor kernels are likely to have the feature enabled, but it will actually be used on relatively few systems. As noted above, there are other places where changing data in pages under writeback can create problems. So the real solution may be some sort of runtime switch - perhaps a filesystem mount option - indicating when stable pages are needed. 

It is also possible that the real problem is somewhere else. Chris Mason [expressed discomfort](<https://lwn.net/Articles/486314/>) with the idea of only using stable pages where they are strictly needed: 

I'm not against only turning on stable pages when they are needed, but the code that isn't the default tends to be somewhat less used. So it does increase testing burden when we do want stable pages, and it tends to make for awkward bugs that are hard to reproduce because someone neglects to mention it. 

According to Chris, writeback latencies simply should not be seen on the scale of multiple seconds; he would like to see some effort put into figuring out why that is happening. Then, perhaps, the real problem could be fixed. But it may be that the real problem is simply that the system's resources are heavily oversubscribed and the I/O queues are long. In that case, a real fix may be hard to come by. 

Boaz Harrosh [suggested](<https://lwn.net/Articles/486315/>) avoiding writeback on the final pages of any files that have been modified in the last few seconds. That might help in the "appending to a log file" case, but will not avoid unpredictable latency resulting from modification of the file at any location other than the end. People have suggested that pages modified while under writeback could be copied, allowing the modification to proceed immediately and not interfere with the writeback. That solution, though, requires more memory (perhaps during a time when the system is desperately trying to free memory) and copying pages is not free. Another option, [suggested](<https://lwn.net/Articles/486316/>) by Ted, would be to add a callback to be invoked by the block layer just before a page is passed on to the device; that callback could calculate checksums and mark the page unwritable only for the (presumably much shorter) time that it is actually under I/O. 

Other solutions certainly exist. The first step, though, would appear to be to get a real handle on the problem so that solutions are written with an understanding of where the latency is actually coming from. Then, perhaps, we can have a stable pages implementation that provides stable data with stable latency in all situations.
