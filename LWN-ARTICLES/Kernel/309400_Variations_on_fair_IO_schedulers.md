---
title: Variations on fair I/O schedulers
url: https://lwn.net/Articles/309400/
date: "December 3, 2008"
category: "Block layer-IO scheduling; Budget fair queuing scheduler; CFQ IO scheduler; Elevator; IO scheduler; Scheduler-Completely fair scheduler"
author: "December 3, 2008 This article was contributed by Goldwyn Rodrigues"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

December 3, 2008

This article was contributed by Goldwyn Rodrigues

An I/O scheduler is a subsystem of the kernel which schedules I/O operations to the various storage devices to get the best possible throughput from those devices. The algorithm is often reminiscent of the algorithm used by elevators when dealing with requests coming from different floors to go up or down. This is the reason I/O scheduling algorithms are also called "elevators." I/O requests are submitted in an order designed to minimize disk head movement (thus minimizing disk seek times), yet guaranteeing good I/O rates. The next request chosen will be dependent on the current disk head position, in order to service the requests quickly, and spend less time seeking, or moving the disk head. However, algorithms may also consider other aspects such as fairness or time guarantees. 

The Completely Fair Queuing (CFQ) I/O scheduler, is one of the most popular I/O scheduling algorithms; it is used as the default scheduler in most distributions. As the name suggests, the CFQ scheduler tries to maintain fairness in its distribution of bandwidth to processes, and yet does not compromise much on the throughput. The elevator's fairness is accomplished by servicing all processes and not penalizing those which have requests far from the current disk head position. It grants a time slice to every process; once the task has consumed its slice, this slice is recomputed and task is added to the end of the queue. The I/O priority is used to compute the time slice granted and the offset in the request queue. 

### The Budget Fair Queuing scheduler

The time-based allocation of the disk service in CFQ, while having the desirable effect of implicitly charging each application for the seek time it incurs, still suffers from fairness problems, especially towards processes which make the best possible use of the disk bandwidth. If the same time slice is assigned to two processes, they may each get different throughput, as a function of the positions on the disk of their requests. Moreover, due to its round robin policy, CFQ is characterized by an O(N) worst-case delay (jitter) in request completion time, where N is the number of tasks competing for the disk. 

The [Budget Fair Queuing (BFQ) scheduler](<http://lwn.net/Articles/306641/>), developed by Fabio Checconi and Paolo Valente, changes the CFQ round-robin scheduling policy based on time slices into a fair queuing policy based on sector budgets. Each task is assigned a budget measured in number of sectors instead of amount of time, and budgets are scheduled using a slightly modified version of the Worst-case Fair Weighted Fair Queuing+ (WF2Q+) algorithm (described in [this paper [compressed PS]](<http://www.cs.cmu.edu/~hzhang/papers/TON-97-Oct.ps.gz>)), which guarantees a worst case complexity of O(logN) and boils down to O(1) in most cases. The budget assigned to each task varies over time as a function of its behavior. However, one can set the maximum value of the budget that BFQ can assign to any task. 

BFQ can provide strong guarantees on bandwidth distribution because the assigned budgets are measured sectors. There are limits, though: processes spending too much time to exhaust their budget are penalized and the scheduler selects the next process to dispatch I/O. The next budget is calculated on the feedback provided by the request serviced. 

BFQ also introduces I/O scheduling within control groups. Queues are collected into a tree of groups, and there is a distinct B-WF2Q+ scheduler on each non-leaf node. Leaf nodes are request queues as in the non-hierarchical case. BFQ supports I/O priority classes at each hierarchy level, enforcing a strict priority ordering among classes. This means that idle queues or groups are served only if there are no best effort queues or groups in the same control group, and best effort queues and groups are served only if there are no real-time queues or groups. As compared to cfq-cgroups (explained later), it lacks per device priorities. The developers however claim that this feature can be incorporated easily. 

### Algorithm

Requests coming to an I/O scheduler fall into two categories, synchronous and asynchronous. Synchronous requests are those for which the application must wait before continuing to send further requests - typically read requests. On the other hand, asynchronous requests - typically writes - do not block the application's progress while they are executed. In BFQ, as in CFQ, synchronous requests are collected in per-task queues, while asynchronous requests are collected in per-device (or, in the case of hierarchical scheduling, per group) queues. 

When the underlying device driver asks for the next request to serve and there is no queue being served, BFQ uses B-WF2Q+, a modified version of WF2Q+, to choose a queue. It then selects the first request from that queue in C-LOOK order and returns it to the driver. C-LOOK is a disk scheduling algorithm, where the next request picked is the one with the immediate next highest disk sector to the current position of the disk head. Once the disk has serviced the maximum sector number in the request queue, it positions the head to the sector number of the request having the lowest sector number. 

When a new queue is selected it is assigned a budget, in disk sector units, decremented each time a request from the same queue is served. When the device driver asks for new requests and there is a queue under service, they are chosen from that queue until one of the following conditions is met: (1) the queue exhausts its budget, (2) the queue is spending too much time to consume its budget, or (3) the queue has no more requests to serve 

On termination of a request, the scheduler recalculates the budget allocated to each process depending on the feedback it gets. For example, for greedy processes which have exhausted their budgets, the budget is increased, whereas if it has been idle for long, its budget is decreased. The maximum budget a process can get is a configurable system parameter (`max_budget`). Two other parameters, `timeout_sync` and `timeout_async`, control the timeout time for consuming the budget of the synchronous and asynchronous queues respectively. In addition, `max_budget_async_rq` limits the maximum number of requests serviced from an asynchronous queue. 

If a synchronous queue has no more requests to serve, but it has some budget left, the scheduler idles (i.e., it tells to the device driver that it has no requests to serve even if there are other active queues) for a short period, in anticipation of a new request from the task owning the queue. 

### Test Results

The developers [compared](<http://algo.ing.unimo.it/people/paolo/disk_sched/>) six different I/O scheduling algorithms: BFQ, [YFQ](<http://www.usenix.org/events/usenix99/full_papers/bruno/bruno_html/node10.html>), SCAN-EDF, CFQ, the Linux anticipatory scheduler, and C-LOOK. They compared a multitude of test scenarios analogous to real-life scenarios, including throughput, bandwidth distribution, latency, and short-term time guarantees. With respect to bandwidth distribution, BFQ can be concluded as the best, and a good algorithm for most scenarios. There were also extensive tests comparing BFQ against CFQ, and the results are available [here](<http://feanor.sssup.it/~fabio/linux/bfq/results.php>). The throughput of BFQ is more or less the same as CFQ, but it scores well in distributing I/O bandwidth fairly among the processes, and displays lower latency with streaming data. 

Using sector budgets instead of time as a factor of granting slice for fair bandwidth distribution is an interesting concept. The algorithm also employs timeouts to terminate requests of "seeky" processes taking too much time to consume their budget and penalizes them. The feedback from current requests help determine future budgets, making the algorithm self-learning. Such tighter bandwidths distribution would be a requirement for systems running virtual machines, or container classes. However, it depends on how BFQ stands the test of time against the tried-and-tested stable CFQ. 

See [the BFQ technical report [PDF]](<http://algo.ing.unimo.it/people/paolo/disk_sched/bfq-techreport.pdf>) for (much) more information. 

### Expanded CFQ

Control Groups provide a mechanism for aggregating sets of tasks, and all their future children, into hierarchical groups. These groups can be allocated dedicated portions of the available resources, or resource sharing can be prioritized within these groups. Control groups are controlled by the cgroups pseudo-filesystem. Once mounted, the top level directory shows the complete set of existing control groups. Each directory made under the root filesystem makes a new group, and resources can be allocated to the tasks listed in the `tasks` file in the individual groups directory. 

Control groups can be used to regulate access to CPU time, memory, and more. There are also several projects working toward the creation of I/O bandwidth controllers for control groups. One of those is the [expanded CFQ scheduler patch for cgroups](<http://lwn.net/Articles/306772/>) by Satoshi Uchida. This patch set introduces a new I/O scheduler called `cfq-cgroups`, which introduces cgroups for the I/O scheduling subsystem. 

This scheduler, as the name suggests, is based on Completely Fair Queuing I/O scheduler. It can take advantage of hierarchical scheduling of processes, with respect to the cgroup they belong to, each cgroup having its own CFQ scheduler. I/O devices in a control group can be prioritized. The time slice given to each hierarchical group per device is a function of the device priority. This helps shaping of I/O bandwidth per group, per device. 

### Usage

To use, cfq-cgroups, select it as a default scheduler at boot by passing `elevator=cfq-cgroups` as a boot parameter. This can also be dynamically changed for individual devices by writing `cfq-cgroups` to `/sys/block/<device>/queue/scheduler`. There are two levels of control: through the cgroups filesystem, for individual groups, and through sysfs, for individual devices. 

Like any other control group, cfq-cgroup is managed through the cgroup pseudo-filesystem. To access the cgroups, mount the pseudo cgroups filesystem: 

```
# mount -t cgroup -o cfq cfq /mnt/cgroup
```

The cgroup directory, by default, will have a file called `cfq.ioprio`, which contains the individual priority on a per-device basis. The time slice received per device per group is a function of the I/O priority listed in `cfq.ioprio`. The `tasks` file represents the list of tasks in the particular group. To make more groups, create a directory in the mounted cgroup directory: 

```
# mkdir /mnt/cgroup/group1
```

The new directories are automatically populated with files, `cfq.ioprio`, `tasks` etc, which are used to control the resources in this group. To add tasks in a group, write the process ID of the task to the `tasks` file: 

```
#echo <pid> > /mnt/cgroup/group1/tasks
```

The `cfq.ioprio` file contains the list of devices and their respective priorities. Each device in the cgroup has a default I/O priority of 3, while the valid values are 0 to 7. To change the priority of a device for the cgroup group1, run: 

```
# echo 2 > /mnt/cgroup/group1/cfq.ioprio
```

This would change the priority of the entire group. To change the I/O priority of a specific device: 

```
# echo 2 sda > /mnt/cgroup/group1/cfq.ioprio
```

To change the default priority while keeping the priority of the devices unchanged: 

```
# echo 4 defaults > /mnt/cgroup/group1/cfq.ioprio
```

The device view shows the list of cgroups and their respective priorities on a per-group basis. This can be changed by: 

```
# echo 2 group1 > /sys/block/sda/queue/iosched/ioprio
```

The device view contain other parameters similar to the CFQ scheduler, such as `back_seek_max` or `back_seek_penalty`, which are specific to the control of the individual device, same as the traditional CFQ. 

### Implementation

The patch introduces a new data structure called `cfq_driver_data` for the control of I/O bandwidth for cgroups. All driver-related data has been moved from the traditional `cfq_data` structure to `cfq_driver_data` structure. Similarly, `cfq_cgroups` is a new data structure to control the cgroup parameters. The organization of data can be assumed as a matrix with `cfq_cgroups` as rows and `cfq_driver_data` as columns, as shown in the diagram below. 

> ![\[cfq-cgroup diagram\]](https://static.lwn.net/images/ns/kernel/cfq-cgroup.png)

At each intersection, there is a `cfqd_data` structure which is responsible for all CFQ related queue handling, so that each `cfq_data` corresponds to one `cfq_cgroup` and `cfq_driver_data` combination. 

When a new cgroup is created, the cfq_data from the parent cgroup is copied into the new group. While inserting new nodes of `cfq_data` into the cgroup, the `cfq_data` structure is initialized with the priority of the `cfq_cgroup`. This way all data of the parent is inherited by the child cgroup, and shows up in the respective files per group in the cgroup filesystem. 

Scheduling of `cfq_data` within the CFQ scheduler is similar to that of the native CFQ scheduler. Each node is assigned a time slice. This slice is calculated according to the I/O priority of the device, using the per-device base time slice. The time slice offset forms the key of the red-black node to be inserted in the service tree. One `cfq_data` entry is picked from the start of the red-black tree and scheduled. Once its time slice expires it is added to the tree again, after recalculation of its time slice offset. So, each `cfq_data` structure acts as a queue node per device, and, within each CFQ data structure, requests are queued as with a regular CFQ queue. 

Both BFQ and cfq-cgroups are attempts to bring a higher degree of fairness to I/O scheduling, with "fairness" being tempered by the desire to add more administrative control via the control groups mechanism. They both appear to be useful solutions, but they must contend with the wealth of other I/O bandwidth control implementations out there. Coming to some sort of consensus on which approach is the right one could prove to be a rather longer process than simply implementing these algorithms in the first place.
