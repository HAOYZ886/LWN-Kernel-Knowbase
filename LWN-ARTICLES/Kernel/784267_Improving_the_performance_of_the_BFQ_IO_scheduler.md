---
title: Improving the performance of the BFQ I/O scheduler
url: https://lwn.net/Articles/784267/
date: "March 29, 2019"
category: "Block layer-IO scheduling"
author: "March 29, 2019 This article was contributed by Paolo Valente"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

March 29, 2019

This article was contributed by Paolo Valente

[BFQ](<https://www.kernel.org/doc/Documentation/block/bfq-iosched.txt>) is a proportional-share I/O scheduler available for block devices since the 4.12 kernel release. It associates each process or group of processes with a weight, and grants a fraction of the available I/O bandwidth proportional to that weight. BFQ also tries to maximize system responsiveness and to minimize latency for time-sensitive applications. Finally, BFQ aims at boosting throughput and at running efficiently. A new set of changes has improved BFQ’s performance with respect to all of these criteria. In particular, they increase the throughput that BFQ reaches while handling the most challenging workloads for this I/O scheduler. A notable example is [DBENCH](<https://dbench.samba.org/>) workloads, for which BFQ now provides 150% more throughput. These changes also improve BFQ’s I/O control — applications start about 80% more quickly under load — and BFQ itself now runs about 10% faster.

Let’s start with throughput improvements and, to introduce them, let’s examine the main cause of throughput loss with BFQ.

#### I/O-dispatch plugging: a necessary evil that lowers throughput

In BFQ, I/O requests from each process are directed into one of a set of in-scheduler queues, called "bfq-queues". Multiple processes may have their requests sent to a shared bfq-queue, as explained in more detail later. A bfq-queue is tagged as being either synchronous or asynchronous if the I/O requests it contains are blocking or non-blocking for the process that issues them, respectively. Read requests tend to be blocking, since the reading process cannot continue without that data; writes are often non-blocking and, thus, asynchronous. 

BFQ serves each bfq-queue, one at a time, with a frequency determined by its associated weight. If "Q" is a synchronous bfq-queue then, to preserve Q’s allotted bandwidth, BFQ cannot switch to serving a new bfq-queue when Q becomes temporarily empty while in service. Instead, BFQ must plug the dispatching of other I/O, possibly already waiting in other bfq-queues, until a new request arrives for Q (or until a timeout occurs).

With fast drives, this service scheme creates a critical shortcoming. Only one core at a time can insert I/O requests into a bfq-queue; a single core may easily be slower to insert requests than a fast drive can serve the same requests. This results in Q often becoming empty while in service. If BFQ is not allowed to switch to another queue when Q becomes empty then, during the servicing of Q, there will be frequent time intervals during which Q is empty and the device can only consume the I/O already submitted to its hardware queues (possibly even becoming idle). This easily causes considerable loss of throughput.

The new changes address this issue in two ways: by improving how BFQ tries to fill the resulting service holes and by reducing the cases where I/O dispatching is actually plugged. We will only look at the main new improvement concerning the second countermeasure.

#### Improving extra-service injection

BFQ implements an I/O-injection mechanism that tries to fill the idle times occurring during the servicing of a bfq-queue with I/O requests taken from other, non-in-service bfq-queues. The hard part is finding the right amount of I/O to inject so as to both boost throughput and not break bandwidth and latency guarantees for the in-service bfq-queue. Before the changes described in this section, the mechanism tried to compute this amount as follows. First, it measured the bandwidth enjoyed by a given bfq-queue when it was not subjected to any extra-service injection. Then, while that bfq-queue was in service, BFQ tried to inject the maximum possible number of extra requests that did not cause the target bfq-queue's bandwidth to decrease too much.

This solution had an important shortcoming: for bandwidth measurements to be stable and reliable, a bfq-queue must remain in service for a much longer time than that needed to serve a single I/O request. Unfortunately, this does not happen with many workloads. With the new changes, the service times of single I/O requests, rather than the bandwidth experienced by a bfq-queue, are measured. Injection is then tuned as a function of how much it increases service times. Single-request service times are meaningful even if a bfq-queue completes few I/O requests while it is in service.

The throughput boost on SSDs is now about 50% on the hardest workloads for BFQ: those that trick BFQ into doing a lot of often unnecessary plugging of I/O dispatches. We’ll see this result on a graph in a moment, combined with the throughput boost provided by the following improvement.

#### Disable queue merging on flash storage with internal queuing

Some applications, such as QEMU, spawn a set of processes that perform interleaved I/O. Such an I/O, taken in isolation (per process), appears random, but it becomes sequential when merged with that of all the other processes in the set. To boost throughput with these processes, BFQ performs queue merging; it redirects the I/O of these processes into a common, shared bfq-queue.

Since they are ordered by I/O-request position, the I/O requests in the shared bfq-queue are sequential. On devices like rotational disks, serving such a sequential I/O definitely boosts throughput compared with serving the random I/O generated by the processes separately. But that is not the case on flash storage devices with internal queuing, which enqueue many I/O requests and serve them in parallel, possibly reordering requests so as to maximize throughput. Thanks to these optimizations and their built-in parallelism, these devices reach the same throughput for interleaved I/O, with or without BFQ reordering. In view of this fact, the new changes disable queue merging altogether on these devices.

As counter-intuitive as it may seem, disabling queue merging actually boosts throughput on these devices; queue merging tends to make many workloads artificially more uneven. Consider the case where one of the bfq-queues in a set of merged bfq-queues has a higher weight than a normal bfq-queue, and where the shared bfq-queue inherits that high weight. I/O dispatching must be plugged while serving the shared bfq-queue, to preserve the higher bandwidth demands of this bfq-queue. In addition, the bfq-queue is filled by several processes, so it tends to remain active for a longer time than normal bfq-queues. In the end, it may force BFQ to perform I/O plugging for a lot of time, hurting overall throughput.

To evaluate the benefits of this improvement, we measured the throughput with DBENCH for the configuration causing the highest throughput loss with BFQ: six clients on a filesystem with journaling, with the journaling daemon enjoying a higher weight than normal processes, and with all other parameters configured as in the DBENCH test in the [MMTests](<https://github.com/gormanm/mmtests>) suite. The throughput grew by about 50% on SSDs. The combined effect of this and the service-injection improvement is shown below: 

> ![\[DBENCH
    throughput\]](https://static.lwn.net/images/2019/bfq/dbench.png)

This plot shows throughput on a PLEXTOR PX-256M5 SSD, compared with the maximum throughput reached by the mq-deadline scheduler (which, in turn achieves the highest throughput among non-BFQ I/O schedulers).

#### Improving responsiveness

When waiting for the arrival of a new I/O request for the in-service bfq-queue, a timeout needs to be set to avoid waiting forever if the processes associated with the bfq-queue have stopped doing I/O. Even if the timeout avoids infinite waiting, the drive is still not fed with new I/O until the timer fires (in the absence of injection). This lowers throughput and inflates latencies. For this reason, the timeout is kept relatively low; 8ms is the current default.

Unfortunately, such a low value may cause a violation of bandwidth guarantees for processes that happen to issue new I/O requests just a little too late. The higher the system load, the higher the probability that this will happen; it is a problem in scenarios where service guarantees matter more than throughput. One important case is when, for example, an application is being started, or is performing some interactive task (such as opening a file). To provide a high level of responsiveness to the application, its I/O requests must be served quickly. This implies that, in the presence of other workloads competing for storage bandwidth, the bfq-queue for the application must be granted a high fraction of the available storage bandwidth. To reach this goal, BFQ tries to automatically detect such queues and raise their weight. But the benefit of this higher weight will be lost in case of late I/O arrivals.

To address this issue, BFQ now places a 20ms lower bound on the dispatch-plugging timeout for weight-raised bfq-queues. This simple change reduces application start-up times under load by up to 80%. This plot show the start-up times of GNOME terminal on a PLEXTOR PX-256M5S drive while ten files are being read sequentially in parallel (10r-seq), or while five files are being read sequentially in parallel, and five more files are being written sequentially in parallel (5r5w-seq). 

> ![\[GNOME terminal startup
times\]](https://static.lwn.net/images/2019/bfq/gnome-terminal.png)

As a reference, start-up times with KYBER are reported too; they are the second lowest start-up times after those with BFQ.

#### Reducing execution time

Handling queue merging costs CPU time, so disabling it reduced the execution time of BFQ, by about 10%, as shown below for an Intel Core i7-2760QM system: 

> ![\[BFQ execution time\]](https://static.lwn.net/images/2019/bfq/exec-time.png)

As a reference, the figure also shows the execution time of mq-deadline, the simplest available I/O scheduler in the kernel. 

To provide more detail: the total times in the figure are the sums of the execution times over three request-processing events: enqueue, dispatch, and completion. So the amortized cost of BFQ, per event, decreased to about 0.6µs, against 0.2µs for mq-deadline.

This improvement reduced the number of CPU and drive configurations for which BFQ cannot currently be used (but mq-deadline can) due to its execution cost. The remaining configurations are those in which switches between user and kernel context, plus 0.2µs of I/O-scheduling overhead, are feasible, but for which an extra 0.4µs per event is not tolerable.

#### Conclusion

Thanks to these new improvements, BFQ seems now to be on par with the other I/O schedulers in terms of throughput, even with workloads that previously "fooled" its heuristics. In contrast, the execution time of BFQ is still higher than that of the other I/O schedulers, but it is not higher than single-request service times on fast drives. The problem now is that BFQ uses a single, per-device scheduler lock. Stay tuned for future work, which will increase parallelism within the BFQ scheduler itself. 

[I wish to thank Alessio Masola for making a very first version of the patch that disables queue merging, Francesco Pollicino for patiently testing the various versions of these patches thousand times, and Mathieu Poirier for carefully revising the first draft of this article.]
