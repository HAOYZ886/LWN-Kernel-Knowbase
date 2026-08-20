---
title: Minisummit reports
url: https://lwn.net/Articles/572063/
date: "October 29, 2013"
category: "Architectures-Arm"
author: "By Jonathan Corbet October 29, 2013 2013 Kernel Summit"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
October 29, 2013

* * *

[2013 Kernel Summit](<https://lwn.net/Articles/KernelSummit2013/>)

Following the usual tradition, the plenary day at the 2013 Kernel Summit started with a series of reports from various kernel minisummits, most of which were held earlier in the week. For some of those meetings, there is reporting that is available elsewhere, so that information will not be duplicated here. 

The events with reports found elsewhere are: 

  * The power-aware scheduling minisummit; this meeting was [covered separately](<https://lwn.net/Articles/571414/>) here on LWN. 

  * The media minisummit; see [Mauro Carvalho Chehab's notes](<https://lwn.net/Articles/571933/>) for details. 

  * The Linux security summit held in New Orleans in September; LWN posted [several articles](<https://lwn.net/Archives/ConferenceIndex/#Linux_Security_Summit-2013>) from that event. 

#### The ARM minisummit

Grant Likely reported from the two-day ARM minisummit held in Edinburgh. This gathering, he said, was "mostly boring"; for the most part, it was "normal engineering stuff". Grant said that it was nice not to have "big [![\[Grant Likely\]](https://static.lwn.net/images/conf/2013/lce-ks/GrantLikely-sm.jpg)](<https://lwn.net/Articles/572130/>) news items" to have to deal with. Notes are promised, but have not been posted as of this writing. 

One of the items discussed was the status of the "single zImage" work, which aims to create a single binary kernel that can boot on a wide variety of ARM hardware. Work is progressing in this area, with support being added for a number of ARM processor types. For the curious, there is [an online spreadsheet](<https://docs.google.com/spreadsheet/ccc?key=0Aj_PQh-9xwkMdHRHYzRzOEhyMEt5b3UtOFIwcHRpWEE&usp=drive_web#gid=0>) showing the current status of many ARM-based chipsets. 

Some time went into the problem of systems with non-discoverable topologies; this is an especially vexing issue in the server area. There was some talk of trying to push the problem into the firmware, but the simple fact is that it is not possible to get the firmware to hide the issue on all systems. 

As anybody who has been unlucky enough to be subscribed to the relevant mailing lists knows, the big issue at the 2013 gathering was the problems with the device tree transition. Grant gave an overview of the discussion as part of his report; more details on the device tree issue came out during [a separate session](<https://lwn.net/Articles/572114/>) later in the day. 

The big problem with device trees is their status as part of the kernel's ABI. As an ABI element, device tree bindings should not change in incompatible ways, but that constraint creates a problem: as the developers learn more about the problem, they need to be able to evolve the device tree mechanism to match. That has led to a situation where driver development has been stalled; the need to create perfect, future-proof device tree bindings has caused work to be hung up in the review process. The number of new bindings is large, while the number of capable reviewers is small. The result is a logjam that is slowing development as a whole. 

There is a plan to resolve some of those issues which was discussed later in the day. In this session, though, Grant raised the question: might device trees be a failed experiment? Should the kernel maybe switch to something else? The alternatives are few, however. The "board file" scheme used in the past has proved to not scale and is an impediment to the single zImage goal. ACPI has its own problems in the ARM space, even if it were to become universally available. One might contemplate the possibility of something completely new, but there are no proposals on the table now. It seems that we are stuck with device trees for now. 

So the ARM developers plan to focus on making things work better in that area. That means that much of the work in the coming year will be aimed at improving processes rather than inventing interesting new technologies. 

[Next: [Git tree maintenance](<https://lwn.net/Articles/572068/>)]
