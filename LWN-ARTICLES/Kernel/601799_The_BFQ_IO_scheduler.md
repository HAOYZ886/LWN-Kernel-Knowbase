---
title: The BFQ I/O scheduler
url: https://lwn.net/Articles/601799/
date: "June 11, 2014"
category: "Block layer-IO scheduling; Elevator; IO scheduler"
author: "By Jonathan Corbet June 11, 2014"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
June 11, 2014

A block-layer I/O scheduler is charged with dispatching I/O requests to storage devices in a way that maximizes throughput while minimizing latencies. The Linux kernel currently includes a few different I/O schedulers, but, for the most part, development in this area has been slow in recent years, with no new schedulers (or major changes to existing schedulers) proposed for some time. That situation has changed with the posting of the ["budget fair queuing" (BFQ) I/O scheduler](<https://lwn.net/Articles/600366/>), which brings some interesting new ideas to this part of the kernel. 

#### Basic BFQ

BFQ, which has been developed and used out of tree for some years, is, in many ways, modeled after the "completely fair queuing" (CFQ) I/O scheduler currently found in the mainline kernel. CFQ separates each process's I/O requests into a separate queue, then rotates through the queues trying to divide the available bandwidth as fairly as it can. CFQ does a reasonably good job and is normally the I/O scheduler of choice for rotating drives, but it is not without its problems. The code has gotten more complex over the years as attempts have been made to improve its performance, but, despite the added heuristics, it can still create I/O latencies that are longer than desired. 

The BFQ I/O scheduler also maintains per-process queues of I/O requests, but it does away with the round-robin approach used by CFQ. Instead, it assigns an "I/O budget" to each process. This budget is expressed as the number of sectors that the process is allowed to transfer when it is next scheduled for access to the drive. The calculation of the budget is complicated (more on this below), but, in the end, it is based on each process's "I/O weight" and observations of the process's past behavior. The I/O weight functions like a priority value; it is set by the administrator (or by default) and is normally constant. Processes with the same weight should all get the same allocation of I/O bandwidth. Different processes may get different budgets, but BFQ tries to preserve fairness overall, so a process getting a smaller budget now will get another turn at the drive sooner than a process that was given a large budget. 

When it comes time to figure out whose requests should be serviced, BFQ examines the assigned budgets and, to simplify a bit, it chooses the process whose I/O budget would, on an otherwise idle disk, be exhausted first. So processes with small I/O budgets tend not to wait as long as those with large budgets. Once a process is selected, it has exclusive access to the storage device until it has transferred its budgeted number of sectors, with a couple of exceptions. Those are: 

  * Normally, a process's access to the device ends if it has no more requests to be serviced. If, however, the last request was synchronous (a read request, essentially), BFQ will let the drive sit idle for a short period to give the process a chance to generate another I/O request. Since the process was probably waiting for the read to complete before generating more I/O traffic, that request comes quite often, and it tends to be contiguous with the last request (or close to it) and, thus, fast to service. It may be counter-intuitive, but idling a drive briefly after synchronous requests tends to increase throughput overall. 

  * There is a maximum period of time allowed for a process to complete its requests. If its I/O load is slow to complete (mostly likely because it consists of random I/O patterns requiring a lot of seeking by the drive), it will lose access to the drive before it has transferred its full budget. In this case, the process will be charged for the full budget anyway to reflect its overall effect on the drive's I/O throughput. 

There is still the question of how each process's budget is assigned. In its simplest form, the algorithm is this: each process's budget is set to the number of sectors it transferred the last time it was scheduled, subject to a systemwide maximum. So processes that tend to do small transfers then stop for a while will get small budgets, while I/O-intensive processes will get larger budgets. The processes with the smaller budgets, which often tend to be more sensitive to latency, will be scheduled more often, leading to a more responsive system. The processes doing a lot of I/O may wait a bit longer, but they will get an extended time slice with the storage device, allowing the transfer of a large amount of data and, hopefully, good throughput. 

#### Bring on the heuristics

Some experience with BFQ has evidently shown that the above-described algorithm can yield good results, but that there is room for improvement in a number of areas. The current posting of the code has, in response, added a set of heuristics intended to push the behavior of the system in the desired direction. These include: 

  * Newly started processes get a medium-sized budget and an increased weight; this allows them to do a fair amount of I/O with minimal delay from the outset. The idea here is to improve application startup time by giving new processes some extra I/O bandwidth to fault their code into memory. The increased weight decays linearly as the process runs. 

  * BFQ's budget calculations, including the maximum allowed budget, are dependent on an estimate of the underlying device's peak I/O rate. The peak I/O rate can vary considerably, though, depending on where the data is located on the disk and how much caching is going on inside the drive. A number of tweaks to the peak-rate calculator try to account for these effects. For example, the observed I/O rate for processes that run out of time without exhausting their budgets is now factored in, even though a timeout of this nature usually indicates that the I/O pattern is random and the drive is not operating at its peak rate. The reasoning is that a timeout can also indicate that the maximum budget value is too large. There is also a low-pass filter used to exclude especially high rate calculations, since those are more likely to be measuring drive caching than actual I/O rates. 

  * The budget calculations themselves have been tweaked. If a process runs out of requests before exhausting its budget, the old response was to lower the budget to the number of requests issued. In the current code, instead, the scheduler will look to see if any of the process's I/O requests are still outstanding; if so, the rate will be doubled on the theory that more requests will be forthcoming when those outstanding requests complete. In the case of a timeout, the budget is, once again, doubled; this tweak is meant to help processes working from slow parts of a drive and to cause processes with truly random I/O patterns to be serviced less frequently. Finally, if a process still has requests outstanding after using its entire budget, it's likely to be an I/O-intensive process; in this case the budget is quadrupled. 

  * Write operations are more costly than reads because disk drives tend to cache the data and signal completion immediately; the actual write to media is done at some later time. That can cause starvation of read requests later on. BFQ tries to account for this cost by counting writes more heavily against the budget; indeed, one write is charged like ten reads. 

  * On drives that can queue multiple commands internally, idling (as described above) can cause the internal queue to empty out, reducing throughput. So BFQ will disable idling on solid-state devices with command queuing. Idling will also be disabled on rotational devices, but only when servicing processes with random I/O patterns. 

  * When multiple processes are operating on the same portion of the disk, it can be better to keep their queues together rather than servicing them separately. Evidently QEMU, which divides I/O among a number of worker threads, is a good example of this type of workload. BFQ includes an algorithm called "early queue merge" that attempts to detect such processes and join their queues together. 

  * BFQ attempts to detect "soft realtime" applications — media players, for example — and boost their weight to help ensure that they experience low latencies. This detection works by looking for a pattern of issuing a set of I/O requests, then going idle (from a disk I/O point of view) for a period of time. Processes that exhibit that pattern will have their weight increased. 

The list of heuristics is longer than this, but one should get the idea: tuning the I/O patterns of a system to optimize for a wide range of workloads is a complex task. From the results posted by BFQ developer Paolo Valente, it seems that a fair amount of success has been achieved. The task of getting this code into the mainline may be just a little bit harder, though. 

#### The path toward merging

If BFQ does have a slow path into the mainline, it will not be because the kernel developers dislike it; indeed, almost all of the comments have been quite positive. The results speak for themselves, but there was also a lot of happiness about how the scheduler has been studied and all of the heuristics have been extensively described and tested. The CFQ I/O scheduler also contains a lot of heuristics, but few people understand what they are or how they work. BFQ appears to be a cleaner and much better documented alternative. 

What the kernel developers do not want to see, though, is the merging of another complex I/O scheduler that tries to fill the same niche as CFQ. Instead, they would like to see a set of patches that evolves CFQ into BFQ, leaving the kernel with a single, improved I/O scheduler. As Tejun Heo [put it](<https://lwn.net/Articles/601808/>): 

Well, it's all about how to actually route the changes and in general whenever avoidable we try to avoid whole-sale code replacement especially when most of the structural code is similar like in this case. Gradually evolving cfq to bfq is likely to take more work but I'm very positive that it'd definitely be a lot easier to merge the changes that way and people involved, including the developers and reviewers, would acquire a lot clearer picture of what's going on in the process. 

Changing CFQ in an evolutionary way would also help when the inevitable performance regressions turn up. Finding the source of regressions in BFQ could be challenging; bisecting a series of changes to CFQ would, instead, point directly to the offending change. 

The BFQ scheduler has been around for a while, and has seen a fair amount of use. Distributions like Sabayon and OpenMandriva ship it, as does CyanogenMod. It seems to be a well-proven technology. All that's needed is some time put into packaging it properly for inclusion into the mainline. Once that has been done, more extensive performance testing can be done. After any issues found there are resolved, this scheduler could replace CFQ (or, more properly, become the future CFQ) in the kernel relatively quickly. 

(See [this paper [PDF]](<http://www.algogroup.unimo.it/people/paolo/disk_sched/bf1-v1-suite-results.pdf>) for a lot more information on how BFQ works).
