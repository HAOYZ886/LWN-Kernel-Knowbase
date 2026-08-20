---
title: An overview of kernel samepage merging (KSM)
url: https://lwn.net/Articles/953141/
date: "November 29, 2023"
category: "Memory management-Kernel samepage merging"
author: "By Jake Edge November 29, 2023 LPC"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
November 29, 2023

* * *

[LPC](<https://lwn.net/Archives/ConferenceByYear/#2023-Linux_Plumbers_Conference>)

In the [Kernel Summit track](<https://lpc.events/event/17/sessions/153/>) at the [2023 Linux Plumbers Conference](<https://lpc.events/event/17/page/198-lpc-2023-overview>) (LPC), Stefan Roesch led a session on [kernel samepage merging](<https://www.kernel.org/doc/html/latest/admin-guide/mm/ksm.html>) (KSM). He gave an overview of the feature and described [some recent changes to KSM](<https://lwn.net/Articles/928510/>). He showed how an application can enable KSM to deduplicate its memory and how the feature can be evaluated to determine whether it is a good fit for new workloads. In addition, he provided some real-world data of the benefits from his workplace at Meta. 

#### KSM basics

The high-level summary of KSM is "very simple": it is simply a scheme to deduplicate anonymous pages by sharing a single copy. It was added to the kernel in 2009, so it is not a new feature, but there has been increased interest in it over the last two years. The original use case was for deduplicating the memory of virtual machines (VMs), but there are other use cases as well. 

[ ![\[Stefan Roesch\]](https://static.lwn.net/images/2023/lpc-roesch-sm.png) ](<https://lwn.net/Articles/953153/>)

In order to do its job, KSM has a kernel thread, `ksmd`, that scans anonymous pages in virtual memory areas (VMAs) that have KSM enabled, which Roesch calls the "candidate pages". It operates in three major phases, using a hash of the contents of the page to quickly compare them against the hashes of other pages to determine if the page is duplicated (or to see if its contents have changed). An `rmap_item` is created for each candidate to track its hash; if a candidate's hash changes frequently, it is not a good choice for deduplication. 

In the second phase, any candidates that have not changed get added to an "unstable" tree; if the candidate is already found to be on the unstable tree, though, it gets moved to the "stable" tree. At that point, other pages with the same contents are switched to use a single page on the stable tree. A copy-on-write (CoW) mechanism is used to ensure that writes to any of the copies are handled correctly. 

There are two ways to add an anonymous page to the candidate set. The "old way" is to use the [`madvise()`](<https://man7.org/linux/man-pages/man2/madvise.2.html>) system call, while the new one uses the [`prctl()`](<https://man7.org/linux/man-pages/man2/prctl.2.html>) system call; the latter was developed by Roesch. Not all memory regions are suitable for KSM, so there are exclusions for regions using DAX, hugetlb, and shared VMAs, he said. 

The `madvise()` mechanism uses a flag, `MADV_MERGEABLE`, to indicate memory regions for KSM to operate on; if it is a compatible region, its pages are added to the candidates. The problem with that approach is that you had to guess which memory regions will benefit because there was no feedback on how well (or poorly) the deduplication is doing for the region. 

The new `prctl()`-based method was added in the 6.4 kernel; the `PR_SET_MEMORY_MERGE` flag can be used to enable KSM for all compatible VMAs in a process. That setting is also inherited when the process forks, so KSM will be enabled for compatible VMAs in any children as well. The `PR_GET_MEMORY_MERGE` flag can be used to query whether KSM is enabled for the process. 

System-wide configuration of KSM is done through the `/sys/kernel/mm/ksm` sysfs interface; there are multiple files in that directory, both for monitoring and configuring the feature. The `run` file is used to enable or disable the feature on the system, `pages_to_scan` determines how many pages are scanned each time `ksmd` wakes up, and `sleep_millisecs` sets how frequently the scans are done. Those latter two govern how aggressively KSM operates. 

For monitoring, there are a few files in the sysfs directory, as well as in the `/proc/PID` directory. In particular, the `/proc/PID/ksm_stat` file has some information about KSM for the process, while some extra KSM information was added to the `smaps` and `smaps_rollups` files for the 6.6 kernel. That information can be used to see which VMAs are benefiting from KSM. 

The monitoring files in `/sys/kernel/mm/ksm` include system-wide measurements of KSM, such as `pages_shared` for the number of pages shared via KSM, `pages_sharing` for the number of references to KSM shared pages (thus how many pages are being deduplicated), `pages_unshared`, which is the number of non-changing pages that are unique, thus unshared, and `pages_volatile` that counts the pages that changed too rapidly. The `pages_scanned` file was added for 6.6 to count the total pages scanned, which can be combined with `full_scans`, the count of scans completed, to determine how much work is being done in the scan phase. 

One challenge is that, prior to the 6.4 kernel, it was not possible to figure out how long the scans were taking. He added some tracepoints to KSM that allow measuring the scan time; `ksm_start_scan` and `ksm_stop_scan` are the two most important tracepoints, but there are a handful of others that are useful for more-specialized investigation. 

#### At Meta

He then turned to how Meta is using KSM. The Instagram web application was suffering from both memory and CPU pressure on older server systems. The workload is characterized by a single controller process and 32 or more worker processes; the number of workers scales based on the size of the system. The workers load their interpreter into memory when they start up and they also share a lot of other data structures that get loaded on demand. 

The Meta engineers thought that KSM would work well for that workload because there is a lot of memory that can potentially be shared. At the time, the only way to enable KSM was via the `madvise()` call. The workers are run in control groups (cgroups) that are started by systemd, so the idea of process-level KSM enabling came up, along with the idea of inheriting that state across `fork()` . 

That is where the `prctl()` flag, which was added for 6.4, came from. At the same time, systemd was [modified](<https://github.com/systemd/systemd/pull/26621/files#>) to add the `MemoryKSM` parameter to enable KSM for a systemd service. The advantage of this approach is that the application code does not need to change at all to take advantage of KSM. 

When he first started testing KSM on the workload, the "results were very disappointing to say the least", Roesch said; there was no real sharing of memory happening. He realized that the default `pages_to_scan` value was set to 100, which is "way too low"; later he noticed that the documentation says that the default is only useful for demo purposes. There were no tracepoints available at the time, either, which made it more difficult to track the problem down. 

It turns out that 4000-5000 is a good compromise value for `pages_to_scan` on the Instagram workload. Other workloads that he has tested require 2000-3000 for that parameter. It is important that people know that the value needs to be changed; looking at the memory savings and the amount of time it takes to do a full scan are good hints for determining the best value. If it is taking 20 minutes to do a full scan, that is an indication the `pages_to_scan` is too low; Meta tries to keep the scan time at around two to three minutes, he said. 

He showed some numbers for a typical workload (which can be seen in his [slides](<https://lpc.events/event/17/contributions/1625/attachments/1320/2649/KSM.pdf>) or the [YouTube video of the talk](<https://youtu.be/VxaAorwL89c?t=22287>)). There were around 73,000 `pages_shared` with 2.1 million references to them (i.e. `pages_sharing`). That means a savings of around 6GB of memory on a 64GB machine, "which is, for us, a huge saving". If you consider the fleet of systems at Meta, that savings multiplies greatly, Roesch said. 

#### Optimizations

Once Meta started looking more closely at the scanning, it was clear that KSM was scanning a huge number of pages, especially during the initial ramp-up as the workers are being started. Even after it reaches something of a steady state, there are lots of pages being repeatedly scanned, but they are unique so they never get shared. That led to the idea of skipping pages as an optimization to reduce CPU usage. 

The "smart scan" optimization feature, which has been [merged for the 6.7 kernel](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=5e924ff54d08>), stores a skip count with each `rmap_item` that governs whether the page is skipped in the processing. The skip count increases (to a maximum of eight scan cycles that will be skipped) each time the page is found to be unique again once its skip count is reached. Smart scan is enabled by default and it reduces the number of pages scanned per cycle by 10-20%. 

An optimization that is being discussed would help tune the number of pages to scan. Right now, that value needs to be set based on the ramp-up time where more than twice the number of pages need to scanned per cycle; once the steady state is reached, the `pages_to_scan` value could be reduced. Other workloads have shown similar behavior, so the ["auto-tune" optimization](<https://lwn.net/ml/linux-kernel/20231028000945.2428830-1-shr@devkernel.io/>) could manage how aggressive the page scans are. The idea would be to identify a target for how long it should take to scan all of the candidate pages, which is what auto-tune would try to optimize. There would also be minimum and maximum CPU usage percentages that would limit the scans as well. 

The results from auto-tune so far are promising. At startup, the `pages_to_scan` gets set to 5000-6000, but that gets reduced to 2500 or even less once the system reaches the steady state. That results in a CPU usage savings of 20-30% for `ksmd`. Configuration using a target scan time and CPU usage limits is more meaningful to administrators, as well, he said. 

#### Evaluating new workloads

The easiest way to enable KSM for an application is by using the `prctl()` flag for the process. That can be done by changing the application itself, using the systemd parameter, or by running the program with an `LD_PRELOAD` library with a function that gets called at program load time. The last option works, but the first two are preferred, he said. 

The next step is to run the program on a representative workload. The `/sys/kernel/mm/ksm/general_profit` file can be consulted to see how much memory is being saved; that measure subtracts out the memory used by KSM itself. The `/proc` files can be consulted for further per-process information as well. 

To get meaningful data, though, it makes sense to rerun the test with different `pages_to_scan` values. How aggressive the page scan should be depends on the workload, so it is important to run the tests long enough to get the full picture. He reiterated that the default value for `pages_to_scan` is not at all adequate, so it will need to be adjusted. 

Often, it is the case that an application has certain VMAs that benefit from KSM and others that do not. The `/proc/PID/smaps` file now has entries for KSM that will help show which VMAs are seeing the most benefit. Once that is known, the `prctl()` call can be removed and separate `madvise()` calls can be made for just those VMAs. One general piece of advice that he had is that smaller page sizes work better with KSM because there is more likelihood of sharing. 

Today, evaluating a new workload for KSM requires running experiments with KSM enabled, but there may be situations where KSM cannot be enabled or these kinds of experiments cannot be run. He has some ideas on ways to evaluate workloads and was looking for feedback on them. One is an in-kernel approach and the other uses the [drgn kernel debugger](<https://lwn.net/Articles/952942/>). 

He has just hacked something together for drgn at this point, which he has not yet released, but the idea is to go through all the VMAs and collect the hashes for the pages, storing them in Python dictionaries. That information can be processed to see how much sharing can be done. It is fairly simple, but is also rather slow; if only a few processes are examined, it is "probably OK", but if the whole system is to be analyzed, "we need to do something else". 

An in-kernel alternative would provide a means to calculate the hashes for the pages so that the sharing could be evaluated. A more advanced scheme would actually maintain the unstable and stable trees but do no merging; that would provide more accurate information about how much sharing can be done, but would be more expensive. These are some ideas he is considering because Meta has other workloads that might benefit from KSM, but running experiments to figure out which would benefit is rather time-consuming. 

There are some security issues to consider with regard to KSM, though "if you control your workload then this is less of a worry". There are known side-channel attacks against KSM, however—he linked to two papers in his slides—so that should be factored into the decision about using KSM. In addition, KSM does not make sense for all workloads; in particular, latency-sensitive workloads are not good candidates for KSM. 

He wrapped up by recounting the KSM changes that entered in kernel in 6.1, 6.4, 6.6, and in the upcoming 6.7, with a nod to the auto-tune feature that will likely come before long. He also credited several of his colleagues for work on the feature and the systemd developers for helping him on that piece of the puzzle. 

Omar Sandoval asked whether auto-tune was being done in the kernel or if it was driven by user space. Roesch said that it was all done in the kernel based on the three parameters (target scan time, CPU min/max). There are default values for those that should be fine for most workloads, but may need tweaking based on the number of pages and the CPU availability. 

Another question was about the CPU and memory overhead for enabling KSM. Roesch said there is a formula in the documentation to calculate the memory overhead, but that it is not much; there are the `rmap_item` entries, which includes the unstable tree that is overlaid on it, plus the stable tree. The CPU overhead depends on how aggressively the scans are done; on a typical Instagram Skylake system during startup "we see up to 60% CPU usage for the `ksmd` kernel background thread", which drops to around 30% in the steady state. 

[I would like to thank LWN's travel sponsor, the Linux Foundation, for assistance with my travel to Richmond for LPC.]
