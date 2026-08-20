---
title: "Measuring (and fixing) I/O-controller throughput loss"
url: https://lwn.net/Articles/763603/
date: "August 29, 2018"
category: "Block layer-IO scheduling"
author: "August 29, 2018 This article was contributed by Paolo Valente"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

August 29, 2018

This article was contributed by Paolo Valente

Many services, from web hosting and video streaming to cloud storage, need to move data to and from storage. They also often require that each per-client I/O flow be guaranteed a non-zero amount of bandwidth and a bounded latency. An expensive way to provide these guarantees is to over-provision storage resources, keeping each resource underutilized, and thus have plenty of bandwidth available for the few I/O flows dispatched to each medium. Alternatively one can use an I/O controller. Linux provides two mechanisms designed to throttle some I/O streams to allow others to meet their bandwidth and latency requirements. These mechanisms work, but they come at a cost: a loss of as much as 80% of total available I/O bandwidth. I have run some tests to demonstrate this problem; some upcoming improvements to the [bfq I/O scheduler](<https://lwn.net/Articles/601799/>) promise to improve the situation considerably. 

Throttling does guarantee control, even on drives that happen to be highly utilized but, as will be seen, it has a hard time actually ensuring that drives are highly utilized. Even with greedy I/O flows, throttling easily ends up utilizing as little as 20% of the available speed of a flash-based drive. Such a speed loss may be particularly problematic with lower-end storage. On the opposite end, it is also disappointing with high-end hardware, as the Linux block I/O stack itself has been [redesigned from the ground up](<https://lwn.net/Articles/552904>) to fully utilize the high speed of modern, fast storage. In addition, throttling fails to guarantee the expected bandwidths if I/O contains both reads and writes, or is sporadic in nature. 

On the bright side, there now seems to be an effective alternative for controlling I/O: the proportional-share policy provided by the bfq I/O scheduler. It enables nearly 100% storage bandwidth utilization, at least with some of the workloads that are problematic for throttling. An upcoming version of bfq may be able to achieve this result with almost all workloads. Finally, bfq guarantees bandwidths with all workloads. The current limitation of bfq is that its execution overhead becomes significant at speeds above 400,000 I/O operations per second on commodity CPUs. 

Using the bfq I/O scheduler, Linux can now guarantee low latency to lightweight flows containing sporadic, short I/O. No throughput issues arise, and no configuration is required. This capability benefits important, time-sensitive tasks, such as video or audio streaming, as well as executing commands or starting applications. Although benchmarks are not available yet, these guarantees might also be provided by the newly proposed [I/O latency controller](<https://lwn.net/Articles/758963/>). It allows administrators to set target latencies for I/O requests originating from each group of processes, and favors the groups with the lowest target latency. 

#### The testbed

I ran the tests with an ext4 filesystem mounted on a PLEXTOR PX-256M5S SSD, which features a peak rate of ~160MB/s with random I/O, and of ~500MB/s with sequential I/O. I used blk-mq, in Linux 4.18. The system was equipped with a 2.4GHz Intel Core i7-2760QM CPU and 1.3GHz DDR3 DRAM. In such a system, a single thread doing synchronous reads reaches a throughput of 23MB/s. 

For the purposes of these tests, each process is considered to be in one of two groups, termed "target" and "interferers". A target is a single-process, I/O-bound group whose I/O is focused on. In particular, I measure the I/O throughput enjoyed by this group to get the minimum bandwidth delivered to the group. An interferer is single-process group whose role is to generate additional I/O that interferes with the I/O of the target. The tested workloads contain one target and multiple interferers. 

The single process in each group either reads or writes, through asynchronous (buffered) operations, to one file — different from the file read or written by any other process — after invalidating the buffer cache for the file. I define a reader or writer process as either "random" or "sequential", depending on whether it reads or writes its file at random positions or sequentially. Finally, an interferer is defined as being either "active" or "inactive" depending on whether it performs I/O during the test. When an interferer is mentioned, it is assumed that the interferer is active. 

Workloads are defined so as to try to cover the combinations that, I believe, most influence the performance of the storage device and of the I/O policies. For brevity, in this article I show results for only two groups of workloads: 

  * **Static sequential** : four synchronous sequential readers or four asynchronous sequential writers, plus five inactive interferers. 
  * **Static random** : four synchronous random readers, all with a block size equal to 4k, plus five inactive interferers. 

To create each workload, I considered, for each mix of interferers in the group, two possibilities for the target: it could be either a random or a sequential synchronous reader. In [a longer version of this article [PDF]](<http://algogroup.unimore.it/people/paolo/pub-docs/extended-lat-bw-throughput.pdf>), you will also find results for workloads with varying degrees of I/O randomness, and for dynamic workloads (containing sporadic I/O sources). These extra results confirm the losses of throughput and I/O control for throttling that are shown here. 

#### I/O policies

Linux provides two I/O-control mechanisms for guaranteeing (a minimum) bandwidth, or at least fairness, to long-lived flows: the throttling and proportional-share I/O policies. With throttling, one can set a maximum bandwidth limit — "max limit" for brevity — for the I/O of each group. Max limits can be used, in an indirect way, to provide the service guarantee at the focus of this article. For example, to guarantee minimum bandwidths to I/O flows, a group can be guaranteed a minimum bandwidth by limiting the maximum bandwidth of all the other groups. 

Unfortunately, max limits have two drawbacks in terms of throughput. First, if some groups do not use their allocated bandwidth, that bandwidth cannot be reclaimed by other active groups. Second, limits must comply with the worst-case speed of the device, namely, its random-I/O peak rate. Such limits will clearly leave a lot of throughput unused with workloads that otherwise would drive the device to higher throughput levels. Maximizing throughput is simply not a goal of max limits. So, for brevity, test results with max limits are not shown here. You can find these results, plus a more detailed description of the above drawbacks, in the long version of this article. 

Because of these drawbacks, a new, still experimental, low limit has been added to the throttling policy. If a group is assigned a low limit, then the throttling policy automatically limits the I/O of the other groups in such a way to guarantee to the group a minimum bandwidth equal to its assigned low limit. This new throttling mechanism throttles no group as long as every group is getting at least its assigned minimum bandwidth. I tested this mechanism, but did not consider the interesting problem of guaranteeing minimum bandwidths while, at the same time, enforcing maximum bandwidths. 

The other I/O policy available in Linux, proportional share, provides weighted fairness. Each group is assigned a weight, and should receive a portion of the total throughput proportional to its weight. This scheme guarantees minimum bandwidths in the same way that low limits do in throttling. In particular, it guarantees to each group a minimum bandwidth equal to the ratio between the weight of the group, and the sum of the weights of all the groups that may be active at the same time. 

The actual implementation of the proportional-share policy, on a given drive, depends on what flavor of the block layer is in use for that drive. If the drive is using the legacy block interface, the policy is implemented by the cfq I/O scheduler. Unfortunately, cfq fails to control bandwidths with flash-based storage, especially on drives featuring command queueing. This case is not considered in these tests. With drives using the multiqueue interface, proportional share is implemented by bfq. This is the combination considered in the tests. 

To benchmark both throttling (low limits) and proportional share, I tested, for each workload, the combinations of I/O policies and I/O schedulers reported in the table below. In the end, there are three test cases for each workload. In addition, for some workloads, I considered two versions of bfq for the proportional-share policy. 

> Name  | I/O policy  | Scheduler  | Parameter for target  | Parameter for each of the four active interferers  | Parameter for each of the five inactive interferers  | Sum of parameters  
> ---|---|---|---|---|---|---  
> low-none | Throttling with low limits | none | 10MB/s | 10MB/s (tot: 40) | 20MB/s (tot: 100) | 150MB/s  
> prop-bfq | Proportional share | bfq | 300 | 100 (tot: 400) | 200 (tot: 1000) | 1700  
  
For low limits, I report results with only none as the I/O scheduler, because the results are the same with kyber and mq-deadline. 

The capabilities of the storage medium and of low limits drove the policy configurations. In particular: 

  * The configuration of the target and of the active interferers for low-none is the one for which low-none provides its best possible minimum-bandwidth guarantee to the target: 10MB/s, guaranteed if all interferers are readers. Results remain the same regardless of the values used for target latency and idle time; I set them to 100µs and 1000µs, respectively, for every group.
  * Low limits for inactive interferers are set to twice the limits for active interferers, to pose greater difficulties to the policy.
  * I chose weights for prop-bfq so as to guarantee about the same minimum bandwidth as low-none to the target, in the same only-reader worst case as for low-none and to preserve, between the weights of active and inactive interferers, the same ratio as between the low limits of active and inactive interferers.

Full details on configurations can be found in the long version of this article. 

Each workload was run ten times for each policy, plus ten times without any I/O control, i.e., with none as I/O scheduler and no I/O policy in use. For each run, I measured the I/O throughput of the target (which reveals the bandwidth provided to the target), the cumulative I/O throughput of the interferers, and the total I/O throughput. These quantities fluctuated very little during each run, as well as across different runs. Thus in the graphs I report only averages over per-run average throughputs. In particular, for the case of no I/O control, I report only the total I/O throughput, to give an idea of the throughput that can be reached without imposing any control. 

#### Results

This plot shows throughput results for the simplest group of workloads: the static-sequential set. 

> ![\[Figure 1\]](https://static.lwn.net/images/2018/iocontrol/fig1.png)

With a random reader as the target against sequential readers as interferers, low-none does guarantee the configured low limit to the target. Yet it reaches only a low total throughput. The throughput of the random reader evidently oscillates around 10MB/s during the test. This implies that it is at least slightly below 10MB/s for a significant percentage of the time. But when this happens, the low-limit mechanism limits the maximum bandwidth of every active group to the low limit set for the group, i.e., to just 10MB/s. The end result is a total throughput lower than 10% of the throughput reached without I/O control. 

That said, the high throughput achieved without I/O control is obtained by choking the random I/O of the target in favor of the sequential I/O of the interferers. Thus, it is probably more interesting to compare low-none throughput with the throughput reachable while actually guaranteeing 10MB/s to the target. The target is a single, synchronous, random reader, which reaches 23MB/s while active. So, to guarantee 10MB/s to the target, it is enough to serve it for about half of the time, and the interferers for the other half. Since the device reaches ~500MB/s with the sequential I/O of the interferers, the resulting throughput with this service scheme would be (500+23)/2, or about 260MB/s. low-none thus reaches less than 20% of the total throughput that could be reached while still preserving the target bandwidth. 

prop-bfq provides the target with a slightly higher throughput than low-none. This makes it harder for prop-bfq to reach a high total throughput, because prop-bfq serves more random I/O (from the target) than low-none. Nevertheless, prop-bfq gets a much higher total throughput than low-none. According to the above estimate, this throughput is about 90% of the maximum throughput that could be reached, for this workload, without violating service guarantees. The reason for this good result is that bfq provides an effective implementation of the proportional-share service policy. At any time, each active group is granted a fraction of the current total throughput, and the sum of these fractions is equal to one; so group bandwidths naturally saturate the available total throughput at all times. 

Things change with the second workload: a random reader against sequential writers. Now low-none reaches a much higher total throughput than prop-bfq. low-none serves much more sequential (write) I/O than prop-bfq because writes somehow break the low-limit mechanisms and prevail over the reads of the target. Conceivably, this happens because writes tend to both starve reads in the OS (mainly by eating all available I/O tags) and to cheat on their completion time in the drive. In contrast, bfq is intentionally configured to privilege reads, to counter these issues. 

In particular, low-none gets an even higher throughput than no I/O control at all because it penalizes the random I/O of the target even more than the no-controller configuration. 

Finally, with the last two workloads, prop-bfq reaches even higher total throughput than with the first two. It happens because the target also does sequential I/O, and serving sequential I/O is much more beneficial for throughput than serving random I/O. With these two workloads, the total throughput is, respectively, close to or much higher than that reached without I/O control. For the last workload, the total throughput is much higher because, differently from none, bfq privileges reads over asynchronous writes, and reads yield a higher throughput than writes. In contrast, low-none still gets lower or much lower throughput than prop-bfq, because of the same issues that hinder low-none throughput with the first two workloads. 

As for bandwidth guarantees, with readers as interferers (third workload), prop-bfq, as expected, gives the target a fraction of the total throughput proportional to its weight. bfq approximates perfect proportional-share bandwidth distribution among groups doing I/O of the same type (reads or writes) and with the same locality (sequential or random). With the last workload, prop-bfq gives much more throughput to the reader than to all the interferers, because interferers are asynchronous writers, and bfq privileges reads. 

The second group of workloads (static random), is the one, among all the workloads considered, for which prop-bfq performs worst. Results are shown below: 

> ![\[Figure 2\]](https://static.lwn.net/images/2018/iocontrol/fig2.png)

This chart reports results not only for mainline bfq, but also for an improved version of bfq which is currently under public testing. As can be seen, with only random readers, prop-bfq reaches a much lower total throughput than low-none. This happens because of the Achilles heel of the bfq I/O scheduler. If the process in service does synchronous I/O and has a higher weight than some other process, then, to give strong bandwidth guarantees to that process, bfq plugs I/O dispatching every time the process temporarily stops issuing I/O requests. In this respect, processes actually have differentiated weights and do synchronous I/O in the workloads tested. So bfq systematically performs I/O plugging for them. Unfortunately, this plugging empties the internal queues of the drive, which kills throughput with random I/O. And the I/O of all processes in these workloads is also random. 

The situation reverses with a sequential reader as target. Yet, the most interesting results come from the new version of bfq, containing small changes to counter exactly the above weakness. This version recovers most of the throughput loss with the workload made of only random I/O and more; with the second workload, where the target is a sequential reader, it reaches about 3.7 times the total throughput of low-none. 

When the main concern is the latency of flows containing short I/O, Linux seems now rather high performing, thanks to the bfq I/O scheduler and the I/O latency controller. But if the requirement is to provide explicit bandwidth guarantees (or just fairness) to I/O flows, then one must be ready to give up much or most of the speed of the storage media. bfq helps with some workloads, but loses most of the throughput with workloads consisting of mostly random I/O. Fortunately, there is apparently hope for much better performance since an improvement, still under development, seems to enable bfq to reach a high throughput with all workloads tested so far. 

[ I wish to thank Vivek Goyal for enabling me to make this article much more fair and sound.]
