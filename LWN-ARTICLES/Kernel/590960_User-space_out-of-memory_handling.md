---
title: "User-space out-of-memory handling"
url: https://lwn.net/Articles/590960/
date: "March 19, 2014"
category: "Memory management-Out-of-memory handling; OOM killer"
author: "March 19, 2014 This article was contributed by David Rientjes"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

March 19, 2014

This article was contributed by David Rientjes

Users of Linux sometimes find themselves faced with the dreaded out-of-memory (OOM) killer, an unavoidable consequence of having overcommitted memory and finding swap completely filled up. The kernel finds itself with no other option than to abruptly kill a process when no memory can be reclaimed. The OOM killer has claimed web browsers, media players, and even X window environments as victims for years. It's very easy to lose a large amount of work in the blink of an eye. 

Occasionally, the OOM killer will actually do something helpful: it will kill a rogue memory-hogging process that is leaking memory and unfreeze everything else that is trying to make forward progress. Most of the time, though, it sacrifices something of importance without any notification; it's these encounters that we remember. One of my goals in my work at Google is to change that. I've recently proposed a patchset to actually give a process a notification of this impending doom and the ability to do something about it. Imagine, for example, being able to actually select what process is sacrificed at runtime, examine what is leaking memory, or create an artifact to save for debugging later. 

This functionality is needed if we want to do anything other than simply kill the process on the machine that will end up freeing the most memory — the only thing the OOM killer is guaranteed to do. Some influence on that heuristic is available through `/proc/<pid>/oom_score_adj`, which either biases or discounts an amount of memory for a process, but we can't do anything else and we can't possibly implement all practical OOM-kill responses into the kernel itself. 

So, for example, we can't force the newest process to be killed in place of a web server that has been running for over a year. We can't compare the memory usage of a process with what it is expected to be using to determine if it's out of bounds. We also can't kill a process that we deem to be the lowest priority. This priority-based killing is exactly what Google wants to do. 

There are two different types of out-of-memory conditions of interest: 

  * **System OOM:** when the system as a whole is depleted of all memory.

  * **Memory controller OOM:** when a control group using the memory controller (a "memcg") has reached its allowed limit.

User-space out-of-memory handling can address OOM conditions for both control groups using the memory controller and for the system as a whole. Either way, the interface is provided by the memory controller since the handler should be implemented in a way that it doesn't care whether it is attached to a memory controller cgroup or not. 

#### A brief tour of the memory controller

The memory controller allows processes to be aggregated together into memcgs and for their memory usage to be accounted together. It also prevents total memory usage from exceeding a configured limit, which provides very effective memory isolation from other processes running on the same system. Processes attached to a memcg may not cause the group as a whole to use more memory than the configured limit. 

When a memcg usage reaches its limit and memory cannot be reclaimed, the memcg is out of memory. This happens because memory allocation within a memcg is done in two phases: the _allocation_ , which is done with the kernel's page allocator, and the _charge_ , which is done by the memory controller. If the allocation fails, the system as a whole is out of memory; if that succeeds and then the charge fails, the memcg is out of memory. 

As your tour guide for the memory controller cgroup, I must first offer a warning: this functionality must be compiled into your kernel. If you're not in control of the kernel yourself, you may find that memcg is not enabled or mounted. Let's check my desktop machine running a common distribution: 

```
$ grep CONFIG_MEMCG /boot/config-$(uname -r)
        CONFIG_MEMCG=y
        CONFIG_MEMCG_SWAP=y
        # CONFIG_MEMCG_SWAP_ENABLED is not set
        # CONFIG_MEMCG_KMEM is not set
```

Ok, good, this kernel has the memory controller enabled. Now let's see if it's mounted: 

```
$ grep memory /proc/mounts
        cgroup /sys/fs/cgroup/memory cgroup rw,memory 0 0
```

It is, at `/sys/fs/cgroup/memory`. If it weren't mounted, we could mount it if we had root privileges with: 

```
mount -t cgroup none /sys/fs/cgroup/memory -o memory
```

At the mount point, there are several control files that can be used to configure the memory controller. This memcg itself is the root memcg — the control group that contains all processes in the system by default. Memcgs can be added by creating directories with `mkdir`, just like any other filesystem. Those memcgs will include all of these control files as well, and you can create children in them as well. 

There are four memcg control files of interest in current kernels: 

  * `cgroup.procs` or `tasks`: a list of process IDs that are attached to this memcg.

  * `memory.limit_in_bytes`: the amount of memory, in bytes, that can be charged by processes attached to this memcg.

  * `memory.usage_in_bytes`: the current amount of memory, in bytes, that is charged by processes attached to this memcg.

  * `memory.oom_control`: allows processes to register `eventfd()` notifications when this memcg is out of memory and control whether the kernel will kill a process or not.

My patch set adds another control file to this set: 

  * `memory.oom_reserve_in_bytes`: the amount of memory, in bytes, that can be charged by processes waiting for OOM notification. Keep reading to see why this is useful and necessary. 

The limit of the root memcg is infinite so that processes attached to it may charge as much memory as possible from the kernel. 

When `memory.use_hierarchy` is enabled, the usage, limit, and reserves of descendant memcgs are accounted to the parent as well. This allows a memcg to overcommit its resources, an important aspect of memcg that we'll talk about later. If a memcg limits its usage to 512 MiB and has two child memcgs with limits of 512 MiB and 256 MiB each, for example, then the group as a whole is overcommitted. 

#### Memory controller out of memory handling

When the usage of a memcg reaches its limit and the kernel cannot reclaim any memory from it or a descendant memcg, it is out of memory. By default, the kernel will kill the process attached to that memcg (or one of its descendant memcgs) that is using the most memory. It is possible to disable the kernel OOM killer by doing 

```
echo 1 > memory.oom_control
```

in the relevant control directory. Now, when the memcg is out of memory, any process attempting to allocate memory will effectively deadlock unless memory is freed. This behavior may seem unhelpful, but that situation changes if user space has registered for a memcg OOM notification. To register for a notification when a memcg is out of memory, a process can use `eventfd()` in a sequence like: 

  * Open `memory.oom_control` for reading.

  * Create a file descriptor for notification by doing `eventfd(0, 0)`.

  * Write "`<fd of open()>` `<fd of eventfd()>`" to `cgroup.event_control`.

The process would then do something like: 

```
uint64_t ret;
        read(<fd of eventfd()>, &ret, sizeof(ret));
```

and this `read()` will block until the memory controller is out of memory. This will only wake up the process when it needs to react to an OOM condition, rather than requiring it to poll the out-of-memory state. 

Unfortunately, there may not be much else that this process can do to respond to an OOM situation. If it has locked its text into memory with `mlock()` or `mlockall()`, or it is already resident in memory, it is now aware that the memory controller is out of memory. It can't do much of anything else, though, because most operations of interest require the allocation of more memory. If this process was a shell, for example, an attempt to run `ps`, `ls`, or even `cat tasks` would stall forever because no memory could be allocated. That leads to an obvious question: how is user space supposed to kill a process if it cannot even get a list of processes? 

The goal of user-space out-of-memory handling is to transition that responsibility to user space so users can do _anything_ they want under these conditions. This is only possible because of memory reserves. With my patchset, it's possible to use the new `memory.oom_reserve_in_bytes` to configure an amount of memory that the limit may be overcharged solely by processes that are registered for out-of-memory notifications. If you run: 

```
echo 32M > memory.oom_reserve_in_bytes
```

then any processes attached to this memcg that has registered for `eventfd()` notifications with `memory.oom_control`, (including notifications from other memcgs), may overcharge the limit by 32MiB. This allows user space to actually do something interesting: read a file, check memory usage, build a list of processes attached to out of memory memcgs, etc. The reserve should only need to be a few megabytes at most for these operations if the process is already locked in memory. 

The user-space OOM handler does not necessarily need to kill a process. If it can free memory in other (usually creative) ways, no kill may be required. Or, it may simply want to create a record for examination later that includes the state of the memcg's memory, process memory, or statistics before re-enabling the kernel OOM killer with `memory.oom_control`. With a reserve, writing to `memory.oom_control` will actually work. 

The memcg remains out of memory until the user-space OOM handler frees some memory (including the memory taken from the reserve), it re-enables the kernel out-of-memory killer, or makes memory available by other means. 

One possible "other means" would be to increase the memcg limit. If top-level memcgs represent individual jobs running on a machine, it's usually advantageous to set the memcg limit for each to be less than the full reservation for that job. The kernel will aggressively try to reclaim memory and push the memcg's usage below its limit before finally declaring it to be out of memory as a last resort. Then, and only then, systems software can increase the limit of the memcg if there is memory available on the system. Don't worry, this job would become the first process killed if the system is out of memory and there's a system user-space OOM handler which we'll describe next! 

It's important that the out-of-memory reserve is configured appropriately for the user-space OOM handler. If an OOM handler is dealing with out-of-memory conditions in other memcgs, the memcg that the OOM handler is attached to is overcharged. If there is more than one user-space OOM handler attached, then `memory.oom_reserve_in_bytes` must be adjusted accordingly depending on the maximum memory usage that is allocated by those handlers. 

#### System out of memory handling

If the entire system is out of memory, then no amount of memory reserve granted by a memcg, including the root memcg, will allow a process to allocate more. In this case, it isn't the _charge_ to the memcg that is failing but rather the _allocation_ from the kernel. 

Handling this situation requires a different type of reserve implementation in the kernel: an amount of memory set aside by the memory-management subsystem that allows user-space out-of-memory handlers to allocate in system OOM conditions when nobody else can. A per-zone reserve is nothing new: the `min_free_kbytes` sysctl knob has existed for years; it ensures that some small amount of memory is always free so that important allocations, such as those that are needed for reclaim or are required by exiting processes to free their own memory, will succeed. The user-space OOM handling reserve is simply a small subset of the `min_free_kbytes` reserve for system out of memory conditions. 

The reserve would be pointless, however, if the kernel out-of-memory killer stepped in and killed something itself. Without the patchset, the OOM killer cannot be disabled for the entire system; the patchset makes it possible to disable the system OOM killer just like you can disable the OOM killer for a memcg. This is done via the same interface, `memory.oom_control`, in the root memcg. 

Access to the reserve is implemented immediately before the kernel out-of-memory killer is called. We do a check with a new per-process flag, `PF_OOM_HANDLER` to determine if this process is waiting on an OOM notification. If it is, and the process is attached to the root memcg, then the kernel will try to allocate below the per-zone minimum watermarks. If the reserve is configured correctly, this effectively guarantees memory to be available for user space to handle the condition. Since the per-process flag is used in the page allocator's slow path, there is no performance downside to this feature: it will simply be a conditional for all processes that aren't handling the out of memory condition. 

An important aspect of this design is that the interface for handling system out-of-memory conditions and handling memory controller out-of-memory conditions is the same. User space should not need to have a different implementation depending on whether it is running on a system unconstrained by memcg or whether it's attached to a child memcg. The user-space OOM handler does not need to be changed in any way: if it's attached to the root memcg, it will handle system out-of-memory conditions and if it's attached to a descendant memcg, it will handle memcg out-of-memory conditions. 

Earlier, we talked about the hierarchical nature of memcg and how it's possible to overcommit memory in child memcgs. This is the same at the top level: it's possible for the sum of the memcg limits of all of the root memcg's immediate children to exceed the amount of system memory. In this case, the memcg out-of-memory reserve is useless for the handling of system OOM conditions since the the memcg has not reached its limit, the _charge_ would succeed but the _allocation_ fails first. 

In configurations such as this, the system-level OOM killer may want to do priority-based killing. Rather than simply killing the process using the most memory on the system, which is the heuristic used by the kernel OOM killer, it may want to sacrifice the lowest priority process depending on business goals or deadlines. Top-level memcgs represent individual jobs running on the machine each with their own limit and priority. Given a memory reserve, it's trivial to kill a process from within the lowest priority memcg. This is exactly what Google wants to do. 

It is also possible to give those jobs the same type of control. If system-level software does a `chown` so that memcg control fields are configurable (except for the limit or reserve, of course) by the job attached at the top level, then the job may create its own child memcgs and enforce its own out-of-memory policy. It may even overcommit its child memcgs so the sum of their limits exceeds its own limit. When the job's limit is reached and the out of memory notification is sent, it may effect a policy as if it were a system out-of-memory condition: the interface is exactly the same. In this way, the top-level memcgs are virtualized environments as if all system memory was bounded by its limit. 

#### Will it be merged?

When this idea has been proposed in the past, there has been some controversy as to whether the kernel really wants to start making a commitment to adding another memory reserve to the kernel. Some have suggested that it is a large maintenance burden to support such a feature and that the number of people who actually want to use it outside of Google is very small. 

Google depends on user-space out-of-memory handling to effect a policy beyond the kernel default of selecting the largest process and killing it. The choice is important especially when talking about one of the most aggressive policies you'll find in Linux: the immediate termination of a process that hasn't necessarily done anything wrong. I believe that making something that is extremely difficult or impossible to achieve and empowers users to have control over such an important thing as process termination is worthwhile. 

#### Future development

In the future, it will be possible to release a library that handles all of the above implementation details behind the scenes and allows users to implement their own `HandleOOM()` function. Such a library could also provide implementations for some of the common actions that a user-space OOM handler may do: read the list of processes on the system or attached to an OOM memory controller, check the memory usage of a particular process, etc. 

It is also possible to replace the disabling of the OOM killer, either memcg or system OOM killer, with an out-of-memory delay. For example, another memcg control file could be added to allow user space a certain amount of time to respond and make memory available before the kernel steps in itself. This could be useful if the user-space OOM handler is buggy or has allocated more than its reserve allows; adding a delay isn't as dangerous as disabling the system OOM killer outright. 

This would also allow your favorite Linux distribution to ship with an out of memory handler that could pop up a window and allow the user to select how to proceed or to diagnose the issue further. This saves your important document or presentation that you've been working hard on by not immediately killing something out from under you for that MP3 you just started playing. 

User-space out-of-memory handling is a powerful tool that will give users and administrators more flexibility in controlling their systems and keep important processes running when they need to be. I've described some motivations for Google; others may use it for something completely different. The functionality can be used by anyone for their own needs exactly because the power is in user space.
