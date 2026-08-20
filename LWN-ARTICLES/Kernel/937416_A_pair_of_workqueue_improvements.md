---
title: A pair of workqueue improvements
url: https://lwn.net/Articles/937416/
date: "July 7, 2023"
category: "Releases-6.5; Workqueues"
author: "By Jonathan Corbet July 7, 2023"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
July 7, 2023

Over the years, the kernel has developed a number of deferred-execution mechanisms to take care of work that cannot be done immediately. For many (or most) needs, the [workqueue subsystem](<https://www.kernel.org/doc/html/latest/core-api/workqueue.html>) is the tool that developers reach for first. Workqueues [took their current form](<https://lwn.net/Articles/355700/>) over a dozen years ago, but that does not mean that there are not improvements to be made. Two sets of patches from Tejun Heo show the pressures being felt by the workqueue subsystem and the solutions that are being tried — with varying degrees of success. 

In normal usage, each subsystem creates its own workqueue (with [`alloc_workqueue()`](<https://elixir.bootlin.com/linux/v6.4.1/source/include/linux/workqueue.h#L390>)) to hold work items. When kernel code needs to defer a task, it can fill in a [`work_struct` structure](<https://elixir.bootlin.com/linux/v6.4.1/source/include/linux/workqueue.h#L98>) with the address of a function to call and some data to pass to that call. That structure can be passed, along with the target workqueue, to a function like [`queue_work()`](<https://elixir.bootlin.com/linux/v6.4.1/source/include/linux/workqueue.h#L480>), and the workqueue mechanism will call the function at some future time. The call is made in process context, meaning that work items can block if need be. There is, of course, a long list of variants to `queue_work()`, and a number of ways in which workqueues themselves can be created, but the core functionality — call a function in process at a later time — remains the same. 

Once upon a time, each workqueue had one or more kernel threads associated with it. As long as there were work items on a queue, the threads would remove and execute those items. The problem with this implementation is that the kernel contains a large number of workqueues, and they can end up processing a lot of work items. That resulted in systems containing many worker threads, all competing with each other. 

The "concurrency-managed workqueue" mechanism found in current kernels, also created by Heo, was designed to address these problems. Workqueues no longer have dedicated kernel threads associated with them; instead, a globally managed set of threads runs items from all workqueues. The workqueue subsystem tries to have exactly one work item running on each CPU at any given time — if, of course, there are that many items in need of execution. Once one work item completes, another is dispatched in its place. 

There is one other complication: since work items are allowed to block, any given work item could be "running" but not actually runnable for long periods of time; that could result in a CPU going idle while there are other work items waiting to be run. The workqueue mechanism handles this case by arranging to be notified whenever one of its worker items blocks. When that happens, another work item will be dispatched (with another thread created to run it, if needed) so that the CPU remains busy. Once the blocked worker wakes up, the workqueue core will notice and stop dispatching items while that worker runs. 

#### Detecting CPU-intensive workers

This mechanism handles the case where a work item blocks, but there is another potentially problematic case. If a work item runs for a long time, it will block any others from running on the same CPU, leading to the starvation of other work items. There is a flag (`WQ_CPU_INTENSIVE`) that can be set when a workqueue is created to indicate that the work items running there may run for a long time; that flag causes the workqueue to be run outside of the normal concurrency-management mechanism so that it doesn't block other workers. Developers are often surprised by which parts of their code take the most CPU time, though, so it is easy to not remember to set that flag when creating a workqueue. As a result, kernel developers occasionally find themselves tracking down performance problems created by CPU-hog work items. 

[This patch set](<https://lwn.net/ml/linux-kernel/20230518030033.4163274-1-tj@kernel.org/>) provides a relatively simple solution to this problem. The workqueue core will, on a regular basis, look at which workers are running on each CPU. If any given worker is found to have run without blocking for a long time (defined as 10ms by default), it will be marked as being CPU-intensive and taken out of the concurrency-management regime, allowing other workers to run. There is an option to have the kernel report work functions that repeatedly are marked in this way; developers can use that information to mark the workqueue from which they are run appropriately. 

This new machinery also makes it relatively easy to track how much CPU time each work item is using. This information has been [made available](<https://git.kernel.org/linus/8a1dd1e547c1>) to user space, allowing developers to see how much time their workqueues are consuming. 

This work was pulled into the mainline during the 6.5 merge window. 

#### Binding unbound workqueues

The discussion to this point has ignored the existence of unbound workqueues, which are created with the `WQ_UNBOUND` flag. These workqueues are documented as running ""workers which are not bound to any specific CPU"", and which are not part of the above-described concurrency-management regime. Unbound workqueues are described as being suited for CPU-intensive tasks that are better managed by the CPU scheduler. 

In practice, as described in [this patch set](<https://lwn.net/ml/linux-kernel/20230519001709.2563-1-tj@kernel.org/>), unbound workqueues have not been fully unbound for a while; instead, the workqueue mechanism tries to contain them within a NUMA node. That increases the locality of the workqueue, improving performance. However, it seems that, on current CPUs, NUMA affinity is not enough. A single NUMA node might now contain multiple L3 caches; spreading work across a node will thus spread it across multiple caches, losing some of the locality that NUMA affinity was meant to produce. This has led to a number of complaints about workqueue performance. 

Fixing this problem, it seems, is not easy, and Heo has concluded that ""there is not going to be one good enough behavior for everyone"". So, instead, the patch set creates three new parameters that can be set for each workqueue: 

  * The "affinity scope", describing the boundaries that should be applied to an unbound workqueue. There are five possible values, binding queues to a single CPU, to a CPU and its siblings, to all CPUs sharing the same L3 cache, to a NUMA node, or to the system as a whole. The NUMA binding matches current workqueue behavior. 
  * The "affinity strictness": how strongly the workqueue is bound to its given scope. With strict affinity, work items cannot run outside of their scope. With non-strict affinity, work items will be started within their scope, but the scheduler will be able to move them outside if that improves the performance of the system overall. 
  * "Localization": if set, work items are always started on the CPU where they were queued; after that, they can be moved as described by the scope and strictness parameters. 

Heo included some benchmarks showing the effects of various combinations of parameters. Changing the localization parameter has proved not to be helpful, and he suggested that it may eventually be dropped from the series. The others gave some small gains depending on the specific workload being run. The overall picture is less than fully clear or, as Heo put it: ""The tradeoff between efficiency gains from improved locality and bandwidth loss from work-conservation deficit poses a conundrum"". 

Linus Torvalds initially [responded](<https://lwn.net/ml/linux-kernel/CAHk-=whA2ztAcVrgsqj39j30LJYhjBSkk6Dju6TY16zGpXpkZQ@mail.gmail.com/>) that this work looked overly focused on throughput while ignoring latency, which he regards as being more important. He was later [convinced](<https://lwn.net/ml/linux-kernel/CAHk-=whbP8BjGyGyXcSKi32orb+1+cHSC2HoVAMNVKwmbq8pSg@mail.gmail.com/>) by Heo, though, that this work could improve both throughput and latency. Brian Norris, who is one of the developers reporting performance problems with current kernels, [had the changes tried](<https://lwn.net/ml/linux-kernel/ZIewlkGJJJUXPFL0@google.com/>) but failed to note any performance improvements — results that Heo [found mystifying](<https://lwn.net/ml/linux-kernel/ZJNMk9oSp1_IYXLU@slm.duckdns.org/>). Torvalds [suggested](<https://lwn.net/ml/linux-kernel/CAHk-=wgXoyxy99HnEFcvf+eUZAS5=ekWt_y84LC3P+0osxh6Jw@mail.gmail.com/>) that the problem might be a bug elsewhere in the workqueue code. 

As of this writing, these workqueue performance problems remain unresolved. It is thus not surprising that this set of patches was not pushed for the 6.5 release. Developers are going to have to dig deeper to figure out why some current system architectures are creating performance problems for workqueues.
