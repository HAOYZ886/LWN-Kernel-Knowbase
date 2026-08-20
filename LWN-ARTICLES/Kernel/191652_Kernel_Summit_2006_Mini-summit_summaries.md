---
title: "Kernel Summit 2006: Mini-summit summaries"
url: https://lwn.net/Articles/191652/
date: "July 18, 2006"
category: "Devicescape stack; Networking-Wireless; Wireless extensions"
author: 
---

[2006 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2006/>) on LWN.net.   
---  
As the complexity of the kernel grows, there is an increasing use of "mini-summits" to talk about specific subsystems. Each of these events typically lasts two or three days - as long as the entire primary summit - and covers its target area in far greater depth. This session was an opportunity for the organizers of recent mini-summits to discuss what was accomplished. 

Matthew Wilcox covered the storage summit, recently held in Vancouver. The presentation, unfortunately, was difficult to hear, so few details will be recounted here. [These notes by Craig Thomas](<http://lists.osdl.org/pipermail/storage-summit-2006/2006-June/000101.html>) have much more information about the summit. 

Matthew mentioned the new block layer concept of "request groups," but didn't get into what they actually do. There was some discussion of how I/O barriers and multipath I/O tend not to work well together - it is too easy for operations to be reordered when they get to the device by different paths. There was some discussion of the difficulty of using sysfs for non-trivial driver control operations; there is just no way to perform a multi-parameter, transactional operation in the sysfs context. There was also some talk about how a new SCSI maintainer might be chosen, should James Bottomley hang up his bow tie and move on; there was agreement among the vendors that having the SCSI maintainer employed by a competitor would not be a problem. 

There was a digression into the problem of certain CDROMs which will not properly report the true length of a disk until the kernel attempts to read past the end. Currently, the result can be the loss of the last few (512-byte) blocks on the disk. There are feasible ways of working around the problem, however, so a solution should not be long in coming. 

John Linville discussed the wireless networking summit. He suggested that interested people read [LWN's coverage of the summit](<http://lwn.net/Articles/179305/>) for the details from that event. Since then, a number of things have happened, including the merging of the zd1211 driver for 2.6.18. 

Work in integrating the Devicescape stack continues, but a few glitches remain. The stack is still not entirely SMP safe, a shortcoming which must be fixed before Devicescape can be merged. While this stack works well with adaptors requiring software MAC support, it is less well suited to smarter devices with hardware MAC controllers. Meanwhile, participation from Devicescape employees has fallen off sharply, and some driver developers are concluding that this stack isn't quite as nice to work with as they had originally thought. Still, integration work continues, and this stack should find its way into the mainline eventually. 

The wireless extensions API took some grief during the discussion. This API has never been entirely popular (though it has achieved its goal of unifying wireless device control for years), and the addition of the netlink interface has seemingly just made things worse. John points out, however, that patches implementing alternative APIs have been in short supply, and, until such a patch comes along, the wireless extensions will not be going anywhere. 

Open issues in the wireless networking area include quality of service support \- this topic was discussed in the Wireless Summit report, and has not changed much since then. There is also the problem of reverse engineering of wireless hardware and just how careful the developers have to be to avoid legal problems. Some developers feel that the bar is being set too high, but Alan Cox recounted the story of the Philips webcam driver and asserted that extreme care remains necessary. Getting into legal trouble with the wrong company could lead to extensive problems for the kernel and those who work with it. 

Arjan Van de Ven talked briefly about the filesystems summit, noting that disks are getting cheaper, bigger, slower, and less reliable. For details, however, he suggested reading [the summary published by LWN](<http://lwn.net/Articles/190222/>). Arjan expressed an interest in getting together with developers interested in working on a next-generation filesystem with a five-year timeline, but this idea was not pursued further here. 

Hugh Dickins covered the memory management summit, held immediately prior to the Kernel Summit. There, the developers talked about fancy new page replacement schemes like [CLOCK-Pro](<http://lwn.net/Articles/147879/>). For now, however, there is more interest in fixing glitches with the current page replacement code than in switching to something entirely new. Some of the infrastructure required by newer page replacement algorithms - dirty page tracking in particular - can be useful in characterizing the performance of the current code, however. 

The shared page table patch was discussed. As has been the case for some years, this patch is still not quite ready for prime time. Hugh noted that RSS accounting remained an "extremely irritating" issue. There are some conflicts between shared page tables and efforts to make Linux more secure \- address space randomization, for example. In the end, the case for shared page tables still has not been fully made. 

Dealing with memory fragmentation was a mini-summit topic, with long-suffering Mel Gorman's work being sent back for another rewrite. Write throttling is an ongoing problem; the current code throttles all writers to a congested drive, rather than just going after the heavy users. The CFQ I/O scheduler helps with this problem, however. The out-of-memory handler needs work, as always, but memory management hackers are "a bit ashamed" to be seen in the vicinity of that code. Finally, huge page support is an ongoing issue, with the right way of making huge pages more robust still to be discovered. 

The last update came from Len Brown, who discussed the power management summit - also [covered on LWN](<http://lwn.net/Articles/181888/>). This gathering, he said, turned out to be more of an educational session than a problem-solving exercise. In a sense, there is not much going on in the power management/ACPI area; the specification is not changing, and Linux supports most of its features at this point. The bulk of the work is going into fixing bugs and making the code more solid. To that end, a test suite has been developed; it currently contains some 1700 tests for the AML interpreter. This suite is expected to be released as free software in the very near future. 

Len gave a quick preview of his OLS talk on battery life. He ran some tests, and determined that Windows XP outperforms Linux with regard to power consumption. An idle laptop running XP runs down its battery in 290 minutes, while Linux only lasts 240 minutes. For the laptop in question, that means that Linux required 2.3 more watts of power to sit there doing nothing. There was also a large difference when playing back a DVD; Windows, it seems, performs aggressive readahead so that it can power down the drive, while Linux has no such capability. Powering down USB devices is also a problem - adding a USB mouse to an idle laptop takes 30 minutes off the battery life. In summary, there is a fair amount of low-hanging fruit waiting for those who would like to improve power consumption on Linux systems. 

  * [Next: Kernel quality and development process](<https://lwn.net/Articles/191650/>).
