---
title: Improving performance with SCHED_EXT and IOCost
url: https://lwn.net/Articles/966618/
date: "April 1, 2024"
category: "BPF-CPU scheduling; Control groups-IO bandwidth controllers; Scheduler-Extensible scheduler class"
author: "By Daroc Alden April 1, 2024 SCALE"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Daroc Alden**  
April 1, 2024

* * *

[SCALE](<https://lwn.net/Archives/ConferenceIndex/#Southern_California_Linux_Expo-2024>)

At [SCALE](<https://www.socallinuxexpo.org/scale/21x>) this year Dan Schatzberg and Tejun Heo, both from Meta, gave back-to-back talks about some of the performance-engineering work that they do there. Schatzberg presented on the [extensible BPF scheduler](<https://lwn.net/Articles/922405>), which has been discussed extensively on the kernel mailing list. Heo presented on IOCost — a control group (cgroup) I/O controller optimized for solid-state disks (SSDs) — and the benchmark suite that is necessary to make it work well on different models of disk. 

#### Scheduling with BPF

For many years, Linux used the [ Completely Fair Scheduler](<https://lwn.net/Articles/230574>) (CFS) to decide which tasks should run on which CPUs at any given time. In kernel version 6.6, the default scheduler changed to the [Earliest Eligible Virtual Deadline First](<https://lwn.net/Articles/925371>) (EEVDF) scheduler. Schatzberg presented what is, in his view, a serious problem with both of these schedulers: iteratively improving the design is difficult, because trying out a new scheduler policy requires rebuilding the kernel and rebooting. He presented the BPF-based scheduler as a way to work around this problem — a proposal that has previously been the subject of much debate in the kernel community. 

[ ![\[Dan Schatzberg\]](https://static.lwn.net/images/2024/Dan_Schatzberg-largerfuzz.png) ](<https://lwn.net/Articles/967162>)

The core idea of a scheduler is simple — "the scheduler's job is just sharing the core" — but implementing a performant scheduler can make things "very complicated very quickly". Schatzberg called out the increasing prevalence of heterogeneous computing systems as a particular source of complexity. There are several attributes of a good scheduler that people might care about, such as fairness, optimization, low overhead, and general applicability. Finding a scheduling policy that is best for a given workload therefore needs to involve a lot of performance measurements and experimentation. Once a scheduling policy has been found, though, using it requires maintaining a scheduler out-of-tree currently. Schatzberg's employer has a policy of trying to upstream as many kernel changes as possible, to reduce its maintenance burden and make use of recent kernels. Upstreaming patches for a constantly changing workload-specific scheduler would be difficult. 

Therefore, Schatzberg and others have been working on an extensible scheduler (`SCHED_EXT`) "allowing you to implement scheduling policies [...] as BPF programs at runtime". Experimenting with a scheduling policy using `SCHED_EXT` involves writing a BPF program, compiling it, and loading it into the running kernel. When changing to a new scheduler, users "don't even need to stop the running workload", which makes experimentation much faster. 

Schatzberg spent some time explaining the work that has gone into making this kind of experimentation safe. Since the scheduler is written in BPF, the BPF verifier ensures it cannot crash or break the running kernel. Furthermore, there is a kernel watchdog in `SCHED_EXT` that will automatically kick a BPF scheduler out if it has failed to schedule a task for a configurable amount of time. So even a broken BPF scheduler that refuses to schedule anything won't permanently hang a machine. 

#### Writing a scheduler

Schatzberg then gave an explanation of how to actually write a scheduler using `SCHED_EXT`. The general idea is that the BPF program implements an [operations structure](<https://lwn.net/Articles/811631>) full of callbacks that the kernel will call during scheduling. The structure that BPF sets up with these callbacks also contains various configuration options, such as the name of the scheduler and the desired watchdog timeout. Three callbacks contain the most vital parts of the scheduler: 

```
s32 (*select_cpu)(struct task_struct *p, s32 prev_cpu, u64 wake_flags);
    
        void (*enqueue)(struct task_struct *p, u64 enq_flags);
    
        void (*dispatch)(s32 cpu, struct task_struct *prev);
```

All three of these have default implementations, but "any sufficiently complicated scheduler will probably implement these three". 

To actually implement these callbacks, "you need to understand dispatch queues", which are data structures that contain a list of runnable tasks. By default, each CPU has its own dispatch queue that it automatically takes tasks from, but the scheduler can create as many auxiliary dispatch queues as it needs. `SCHED_EXT` handles all the required locking around performing operations on these queues. 

With that understanding in mind, Schatzberg showed the code to make a simple global first-in first-out scheduler. This is a naive scheduling policy, but Schatzberg noted: "This actually outperforms CFS for some of our production workflows" because getting a task onto an idle core as quickly as possible has some benefits. The example scheduler enqueues tasks by adding them to a global dispatch queue, and then dispatches tasks from that queue when a CPU runs out of work on its own queue. 

```
s32 BPF_STRUCT_OPS_SLEEPABLE(mysched_init)
        {
          scx_bpf_create_dsq(/* queue id = */ 0, -1);
        }
```

`mysched_init()` creates the global dispatch queue that the scheduler uses. 

```
void BPF_STRUCT_OPS(mysched_enqueue, struct task_struct *p, u64 enq_flags)
        {
          scx_bpf_dispatch(p, /* queue id = */ 0, SCX_SLICE_DFL, enq_flags);
        }
```

`mysched_enqueue()` accepts a task from the kernel, and places it directly on the end of the global dispatch queue. 

```
void BPF_STRUCT_OPS(mysched_dispatch, s32 cpu, struct task_struct *prev)
        {
          scx_bpf_consume(/* queue id = */ 0);
        }
```

`mysched_dispatch()` runs when a CPU becomes idle and needs another task. It takes the task from the front of the global dispatch queue and immediately starts it running. 

Schatzberg noted that the [ full implementation](<https://github.com/sched-ext/scx/blob/main/scheds/c/scx_simple.bpf.c>) of the scheduler was slightly longer because it needs to handle tasks such as waking up a CPU that has gone to sleep when there's more work, but emphasized that a complete scheduler could be written in 150 lines, including comments. 

He then gave an example of a slightly less naive scheduler that keeps a separate queue per L3 cache. He pointed out that even this simple optimization raises a lot of questions — when should cores steal work from other L3 caches? When should work be proactively load-balanced across the different queues? Schatzberg said that the real answer is that it depends on the workload, and that developers should experiment using `SCHED_EXT`. "The whole idea is that `SCHED_EXT` lets us experiment with these kinds of policies really quickly". 

He finished off the talk by giving some examples of various schedulers suited to different purposes. Those schedulers, as well as various support tools to make writing BPF-based schedulers easier, are available [ on GitHub](<https://github.com/sched-ext/scx/tree/main>). For the future, he sees the top priority as getting `SCHED_EXT` upstreamed into the kernel — a task that may be difficult since scheduler maintainer Peter Zijlstra [dislikes](<https://lwn.net/Articles/939332/>) the patch series. Schatzberg thought that `SCHED_EXT` could be valuable to users: "We want to build a community around it". He said that there were many features that could be added to make `SCHED_EXT` more usable, but that they were mostly BPF features, and not `SCHED_EXT` features per se. 

#### IOCost

After Schatzberg's talk, Heo described his own performance work, this time in the area of cgroup I/O controllers. [ Cgroups](<https://lwn.net/Articles/603762>) are hierarchical groupings of processes that can have resource limits attached to different nodes in the hierarchy. Heo compared them to a filesystem that contains the processes running on a system. Cgroup controllers have the job of allocating resources to processes within a cgroup, and reacting if they exceed their limits. There are existing cgroup controllers for memory usage, CPU usage, process creation, block device I/O, and more. 

[ ![\[Tejun Heo\]](https://static.lwn.net/images/2024/Tejun_Heo-fuzz.png) ](<https://lwn.net/Articles/967159>)

Heo has been working on a controller for block device I/O called [ IOCost](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/block/blk-iocost.c>). The controller is intended to distribute solid-state disk block operations to different cgroups, ensuring that applications cannot hog the disk's bandwidth. 

He opened by describing why writing an I/O controller is more challenging than writing a controller for memory or CPU. Firstly, the metrics usually used to measure performance of block devices "are not great". For example, specifying the budget of an application in bytes per second or I/O requests per second can result in a limit that is simultaneously too low and too high, because the actual number of bytes per second that a solid-state disk (SSD) can service varies wildly based on several factors. "That makes single-metric based control, based on these numbers, really challenging". 

More problematically, SSDs can be so fast that if the overhead from the cgroup controller is too high, it can noticeably impact the performance of the application. This isn't the case for rotating drives, where it is frequently worth sacrificing some CPU time to optimize disk reads, because the disk latency is so high. That means that existing I/O controllers designed for rotating disks often perform poorly on SSDs. 

And finally, block devices are a shared resource that the entire system needs to use, which makes it easy to accidentally create priority inversions. For example, writing to a journaling filesystem requires writing to a shared journal for the whole filesystem. If the I/O controller delays a low-priority process's write to the journal, all of the high-priority processes that also need to write to the journal are stuck waiting as well. Priority inversion isn't a problem unique to block devices, but it is especially noticeable there since every process on the system contends for the same resource. 

IOCost addresses these challenges in a number of ways, starting by measuring the expense of block operations using "a fairly simple linear model" that is "better than any single metric". It also doesn't try to restrict applications to a numeric limit. Heo pointed out that if you ask an application developer how many I/O operations per second or MB per second they need, "you don't get a good answer. Nobody really knows." So instead, IOCost imposes relative limits — allowing the administrator to say how much I/O each cgroup should receive proportionally. 

To make access decisions quickly enough to not impede SSD performance, IOCost separates its logic into two parts: a planning path that runs every few milliseconds, and an execution path that runs on every request. The planning path handles all of the complex calculations, and then pushes the configuration to the execution path, so that the execution path doesn't need to do much to determine how a request should be handled. This helps keep the latency overhead of the controller low. 

Finally, IOCost integrates with the filesystem and memory-management facilities in the kernel to avoid priority inversions. Throttled I/O requests that are expected to cause priority inversions are run right away, but then charged to the process that exceeded its limits. Heo summarized this approach as "do first, pay later" 

IOCost works well in practice for the use-case for which it was designed. Heo showed the results of some internal tests which demonstrated that IOCost has negligible latency overhead, and perfectly maintains the configured ratio of I/O between two benchmark workloads. IOCost "is fully deployed at Meta, on almost all our machines". He summarized this section with a slide that said "IOCost works well as long as it[']s configured well". 

But that statement conceals a lot of complexity, because it is not easy to configure IOCost well. The linear cost-model that IOCost uses to estimate how expensive a particular I/O request is requires tuning for each model of SSD, because they all behave differently. To solve this problem, Heo and his collaborators created [ `resctl-bench`](<https://github.com/facebookexperimental/resctl-demo?tab=readme-ov-file#resctl-bench>): a scenario-based benchmarking tool that observes the end-to-end behavior of an SSD. It is part of the larger resctl-demo project, which includes resources for various different types of resource control on Linux. `resctl-bench` imitates a workload "similar to what we see in the production environment". Heo also said that it was a major improvement on other benchmarks because: "The bar is fairly low here. We don't have a bar, because nothing works well". 

`resctl-bench` is not easy to set up and run, so Heo also made an installable Linux image that runs the benchmark and automatically generates a report (and configurations for IOCost) based on the result. He noted that "I didn't want to do a live demo, because it takes eight hours", but he did show a series of slides explaining how to use the tool to characterize an SSD and then submit the measurements to [ the database](<https://github.com/iocost-benchmark/iocost-benchmarks>) where he is collecting statistics on different SSD models. He called for interested people to run the benchmark on their own systems, because more data points can make IOCost more performant for everyone. 

One audience member asked whether it was viable to have IOCost learn the parameters it needs on the fly. Heo explained that it wasn't, because of how difficult it is to get the needed feedback from an SSD under normal operation. "It would be really great if you could", he remarked. Another audience member asked whether the benchmark takes into account SSD over-provisioning — the practice of putting more storage into an SSD than its stated capacity, so that the firmware of the disk has spare sectors to replace failing sectors with. Benchmarks that only write a disk's claimed capacity often provide inaccurate results about long-term performance for over-provisioned disks. Heo affirmed that `resctl-bench` does take overprovisioning into account, and that it ends up writing many times the disk's claimed capacity in the course of the benchmark. 

I asked about the plans for IOCost; Heo said "in our fleet, it seems to work well", and that he wanted to see more people using it. He sees IOCost as fairly complete, with the notable exception of how painful it is to configure. He hopes to improve that over time. 

IOCost is already part of the kernel, and can be configured by following [ the documentation](<https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html#io>). The `resctl-bench` [ documentation](<https://facebookmicrosites.github.io/resctl-demo-website/docs/>) explains how to try the benchmark on one's own devices. 

[I would like to thank LWN's travel sponsor, the Linux Foundation, for travel assistance to Pasadena for SCALE.]
