---
title: The creation of the io.latency block I/O controller
url: https://lwn.net/Articles/782876/
date: "March 14, 2019"
category: "Control groups-IO bandwidth controllers"
author: "March 14, 2019 This article was contributed by Josef Bacik"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

March 14, 2019

This article was contributed by Josef Bacik

Sharing a disk between users in Linux is awful. Different applications have different I/O patterns, they have different latency requirements, and they are never consistent. Throttling can help ensure that users get their fair share of the available bandwidth but, since most I/O is in the writeback path, it's often too late to throttle without putting pressure elsewhere on the system. Disks are all different as well. You have spinning rust, solid-state devices (SSDs), awful SSDs, and barely usable SSDs. Each class of device has its own performance characteristics and, even in a single class, they'll perform differently based on the workload. Trying to address all of these issues with a single I/O controller was tricky, but we at Facebook think that we have come up with a reasonable solution. 

Historically, the kernel has had two I/O controllers for control groups. The first, io.max, allows setting hard limits on the bandwidth used or I/O operations per second (IOPS), per device. The second, io.cfq.weight, was provided by the CFQ I/O scheduler. As Facebook has worked on things like [pressure-stall information](<https://lwn.net/Articles/759781/>) and the [version-2 control-group interface](<https://lwn.net/Articles/601840/>), it became apparent that neither of those controllers solved our problem. Generally, we have a main workload that runs, and then we have periodic system utilities that run in the background. Chef runs a few times an hour, updates any settings on the system, and installs packages. The `fbpkg` tool downloads new versions of the application that is running on the system three or four times per day. 

The io.max controller allowed us to clamp down on those system utilities, but made them run unbearably slowly all of the time. Ratcheting up on the throttling just made them impact the main workload too much, so it wasn't a great solution. The CFQ io.cfq.weight controller was a non-starter, as CFQ did not work with the multi-queue block layer, not to mention that just using CFQ in general caused so many problems with latencies that we had turned it off years ago in favor of the deadline scheduler. 

Jens Axboe's [writeback-throttling work](<https://lwn.net/Articles/682582/>) introduced a new way of monitoring and curtailing workloads. It works by measuring the latencies of reads from a disk and, if they exceed a configured threshold, it clamps down on the number of writes that are allowed to go to the disk. This sits above the I/O scheduler, which is important because we have a finite number of requests we can have outstanding for any single device. This number is controlled by the `/sys/block/<device>/queue/nr_requests` setting. We call this the "queue depth" of the device. The writeback-throttling infrastructure worked by lowering the queue depth before allocating a request for incoming write operations, allowing the available requests to be used by reads and throttling the writes as necessary. 

This solution addressed a problem wherein `fbpkg` would pull down multi-gigabyte packages to update the running application. Since the application updates tended to be pushed all at once, we would see global latency spikes as the sudden wave of writes impacted the already running application. 

#### Enter a new I/O controller

Writeback throttling isn't control-group aware and only really cares about evening out read and write latencies per disk. However it has a lot of good ideas, all of which I blatantly stole for [the new controller](<https://lwn.net/Articles/758963/>), which I call io.latency. This controller has to work on both spinning rust and high-end NVMe SSDs, so it needed to have a low overhead. My goal was to add no locking in the fast path, a goal I mostly accomplished. Initially we really wanted both workload protection and proportional control. We have use cases where we want to protect the main workload at all costs, but other use cases where we want to stack multiple workloads and have them play together nicely. Eventually we had to drop that plan and go for workload protection only, and come up with another solution for proportional control. 

With io.latency, one sets a latency threshold for a group. If this threshold is exceeded for a given time period (250ms normally), then the controller will throttle any peers that have a higher latency threshold ![\[control-group hierarchy\]](https://static.lwn.net/images/2019/io.latency.png) setting. The throttling mechanism is the same as writeback throttling: the controller simply clamps down on the queue depth for that particular control group. This throttling only applies to peers in the control-group hierarchy, so in the case shown to the right, for example, if `fast` misses its latency threshold, then only `slow` would be throttled, while `unrelated` would be unaffected. 

The way I accomplish this without locking is to have a cookie that is set in both the parent and its children. If, for example, `fast` misses its target, it decrements the cookie in its parent group (`b`). The next time `slow` submits an I/O request, the controller checks the cookie in `b` against `slow`'s copy of the cookie. If the value has gone down, `slow` decreases its queue depth. If the value has gone up then `slow` would increase its queue depth. 

In the normal I/O path, io.latency adds two atomic operations: one to read the parent cookie and one to acquire a slot in the queue. In the completion path, we only have one atomic operation (to release the queue slot) in the normal case, along with a per-CPU operation to account for the time the I/O took. In the slow case, which occurs every window sample time (that's the 250ms time period mentioned above) we have to acquire a lock in the parent to add up all of the I/O statistics and check if our latencies have missed the threshold. 

Part of io.latency is accounting for the I/O time. Since we care about total latency suffered by the application, we count from the time that each operation is submitted to the time it is completed. This time is kept in a per-CPU structure that is summed up every window period. We found in testing that taking the average latency was good for rotating drives, but for SSDs it wasn't responsive enough. Thus, for SSDs, we have a percentile calculation in place; if the 90th-percentile latencies surpass the threshold setting, then it's time for a less-important peer group to be throttled. 

The final part of io.latency is a timer that fires once each second. Since the controller was built to be mostly lockless, it's driven by the I/O being done. However, if you have the main workload throttling a slower workload into oblivion then ceasing I/O, we will no longer have a mechanism to unclamp the slow group. The periodic timer takes care of this by firing when there's I/O occurring from any source and verifying that the aggrieved group is still doing I/O, otherwise it unclamps everybody so they can go on about their work. 

#### Everything worked perfectly, right?

Unfortunately, the kernel is a large system of interconnected parts, and many of these parts don't like the fact that, suddenly, `submit_bio()` can take much longer to return if the caller is being throttled. We kept running into a variety of different priority inversions that ate up a lot of our time when testing this whole system in production. 

Our test scenario was an overloaded web server with a slow memory leak that was started under the `slow` control group. Generally, what happens is that the `fast` workload will start being driven into memory reclaim and needing to do swap I/O for whatever pages it can get. Pages are attached to their owning control group, which means any I/O performed using those pages is done within the owner's limits. Our high-priority workload was swapping pages owned by a low-priority group, which meant that it was being incorrectly throttled. 

This was easy enough to solve: just add a `REQ_SWAP` flag to the I/O operation and make it so the I/O controller simply let those operations through with no throttling. A similar thing had to be done for `REQ_META` as well, since we could get blocked up on a metadata I/O that the slow group had submitted. However, now the slow group was causing a lot of I/O pressure, but not in a way that caused it to be throttled, since all `REQ_SWAP` I/O is now free. The bad workload was only allocating memory — and never doing I/O — so there was no way to throttle it until it buried the main workload. Once the memory pressure starts to build, the workload's latencies really go through the roof because, for the most part, the main workload is memory intensive, not I/O intensive. 

Another set of infrastructure had to be added to solve this problem. We knew that we were doing a lot of I/O on behalf of a badly behaving control group; we just needed a way to tell the memory-management layer that this group was behaving poorly. To solve this problem I added a congestion counter to the block control-group structure that can be set if a control group is getting a lot of I/O done for free without being throttled. Since we know which control group was responsible for the pages being submitted, we can tag that group as congested, and the memory-management layer will know it needs to start throttling things. 

The next problem we were having was with the `mmap_sem` semaphore. In our workload, there is some monitoring code that does the equivalent of `ps`; it reads `/proc/<pid>/cmdline` which, in turn, takes `mmap_sem`. The other thing that takes `mmap_sem` is the page-fault handler. If tasks performing page faults are being throttled, thus holding `mmap_sem`, and our main workload tries to read a throttled task's `/proc/<pid>/cmdline` file, the main workload would be blocked waiting for the throttled I/O to complete. This meant we had to find a way to do the harsh throttling outside of the path of any possible kernel locking that would cause problems. The `blkcg_maybe_throttle_current()` infrastructure was added to handle this problem. We would add artificial delays to the current task, then, as we return to user space, when we know we aren't holding any kernel locks, we would pause for the given delay to make sure we were still throttled. 

With all of these things in place we had a working system. 

#### Results

Previously, when we would run this memory leak test with no I/O controller in place, the box would be driven into swap and thrash for many minutes until either the out-of-memory killer brought everything down or our automated health checker noticed something was wrong and rebooted the box manually. It takes a while for our boxes to come back up, be integrated back into the cluster, and become ready to accept traffic, so on average there were about 45-50 minutes of downtime for a box with this reproducer. 

With the full configuration in place and [oomd](<https://github.com/facebookincubator/oomd>) monitoring everybody, we'd drop about 10% of our requests per second; then the memory hog would become so throttled that oomd would see it and kill it. This is a 10% drop on an overloaded web server; in normal traffic you'd likely see less or no impact on the overall performance. 

#### Future work

The io.latency controller, along with all of our other control-group work and oomd, currently runs in production on all of our web servers, all of our build servers, and on the messenger servers. It has been stable for a year and has drastically reduced the number of unexpected reboots across those tiers. The next step is to build a proportional I/O controller, to be called io.weight. It's currently in development; production testing will start soon and will likely be posted upstream in the next few months. Thankfully all of the various priority inversions that were found with io.latency have all been fixed, which makes adding new I/O controllers much more straightforward.
