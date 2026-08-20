---
title: Taming the OOM killer
url: https://lwn.net/Articles/317814/
date: "February 4, 2009"
category: "Memory management-Out-of-memory handling; OOM killer"
author: "February 4, 2009 This article was contributed by Goldwyn Rodrigues"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

February 4, 2009

This article was contributed by Goldwyn Rodrigues

Under desperately low memory conditions, the out-of-memory (OOM) killer kicks in and picks a process to kill using a set of heuristics which has evolved over time. This may be pretty annoying for users who may have wanted a different process to be killed. The process killed may also be important from the system's perspective. To avoid the untimely demise of the wrong processes, many developers feel that a greater degree of control over the OOM killer's activities is required. 

### Why the OOM-killer?

Major distribution kernels set the default value of `/proc/sys/vm/overcommit_memory` to zero, which means that processes can request more memory than is currently free in the system. This is done based on the heuristics that allocated memory is not used immediately, and that processes, over their lifetime, also do not use all of the memory they allocate. Without overcommit, a system will not fully utilize its memory, thus wasting some of it. Overcommiting memory allows the system to use the memory in a more efficient way, but at the risk of OOM situations. Memory-hogging programs can deplete the system's memory, bringing the whole system to a grinding halt. This can lead to a situation, when memory is so low, that even a single page cannot be allocated to a user process, to allow the administrator to kill an appropriate task, or to the kernel to carry out important operations such as freeing memory. In such a situation, the OOM-killer kicks in and identifies the process to be the sacrificial lamb for the benefit of the rest of the system. 

Users and system administrators have often asked for ways to control the behavior of the OOM killer. To facilitate control, the `/proc/_< pid>_/oom_adj` knob was introduced to save important processes in the system from being killed, and define an order of processes to be killed. The possible values of `oom_adj` range from -17 to +15. The higher the score, more likely the associated process is to be killed by OOM-killer. If `oom_adj` is set to -17, the process is not considered for OOM-killing. 

### Who's Bad?

The process to be killed in an out-of-memory situation is selected based on its badness score. The badness score is reflected in `/proc/_< pid>_/oom_score`. This value is determined on the basis that the system loses the minimum amount of work done, recovers a large amount of memory, doesn't kill any innocent process eating tons of memory, and kills the minimum number of processes (if possible limited to one). The badness score is computed using the original memory size of the process, its CPU time (utime + stime), the run time (uptime - start time) and its `oom_adj` value. The more memory the process uses, the higher the score. The longer a process is alive in the system, the smaller the score. 

Any process unlucky enough to be in the `swapoff()` system call (which removes a swap file from the system) will be selected to be killed first. For the rest, the initial memory size becomes the original badness score of the process. Half of each child's memory size is added to the parent's score if they do not share the same memory. Thus forking servers are the prime candidates to be killed. Having only one "hungry" child will make the parent less preferable than the child. Finally, the following heuristics are applied to save important processes: 

  * if the task has nice value above zero, its score doubles 

  * superuser or direct hardware access tasks (CAP_SYS_ADMIN, CAP_SYS_RESOURCE or CAP_SYS_RAWIO) have their score divided by 4. This is cumulative, i.e., a super-user task with hardware access would have its score divided by 16. 

  * if OOM condition happened in one cpuset and checked task does not belong to that set, its score is divided by 8. 

  * the resulting score is multiplied by two to the power of `oom_adj` (i.e. `points <<= oom_adj` when it is positive and `points >>= -(oom_adj)` otherwise). 

The task with the highest badness score is then selected and its children are killed. The process itself will be killed in an OOM situation when it does not have children. 

### Shifting OOM-killing policy to user-space

`/proc/_< pid>_/oom_score` is a dynamic value which changes with time, and is not flexible with different and dynamic policies required by the administrator. It is difficult to determine which process will be killed in case of an OOM condition. The administrator must adjust the score for every process created, and for every process which exits. This could be quite a task in a system with quickly-spawning processes. In an attempt to make OOM-killer policy implementation easier, a [name-based solution](<http://lkml.org/lkml/2009/1/12/235>) was proposed by Evgeniy Polyakov. With his patch, the process to die first is the one running the program whose name is found in `/proc/sys/vm/oom_victim`. A name based solution has its limitations: 

  * task name is not a reliable indicator of true name and is truncated in the process name fields. Moreover, symlinks to executing binaries, but with different names will not work with this approach 

  * This approach can specify only one name at a time, ruling out the possibility of a hierarchy 
  * There could be multiple processes of the same name but from different binaries. 

  * The behavior boils down to the default current implementation if there is no process by the name defined by `/proc/sys/vm/oom_victim`. This increases the number of scans required to find the victim process. 

Alan Cox disliked this solution, [suggesting ](<http://lkml.org/lkml/2009/1/12/246>) that containers are the most appropriate way to control the problem. In response to this suggestion, the [oom_killer controller](<http://lkml.org/lkml/2009/1/29/220>), contributed by Nikanth Karthikesan, provides control of the sequence of processes to be killed when the system runs out of memory. The patch introduces an OOM control group (cgroup) with an `oom.priority` field. The process to be killed is selected from the processes having the highest `oom.priority` value. 

To take control of the OOM-killer, mount the cgroup OOM pseudo-filesystem introduced by the patch: 

```
# mount -t cgroup -o oom oom /mnt/oom-killer
```

The OOM-killer directory contains the list of all processes in the file `tasks`, and their OOM priority in `oom.priority`. By default, `oom.priority` is set to one. 

If you want to create a special control group containing the list of processes which should be the first to receive the OOM killer's attention, create a directory under `/mnt/oom-killer` to represent it: 

```
# mkdir lambs
```

Set `oom.priority` to a value high enough: 

```
# echo 256 > /mnt/oom-killer/lambs/oom.priority
```

`oom.priority` is a 64-bit unsigned integer, and can have a maximum value an unsigned 64-bit number can hold. While scanning for the process to be killed, the OOM-killer selects a process from the list of tasks with the highest oom.priority value. 

Add the PID of the process to be added to the list of tasks: 

```
# echo <pid> > /mnt/oom-killer/lambs/tasks
```

To create a list of processes, which will not be killed by the OOM-killer, make a directory to contain the processes: 

```
# mkdir invincibles
```

Setting `oom.priority` to zero makes all the process in this cgroup to be excluded from the list of target processes to be killed. 

```
# echo 0 > /mnt/oom-killer/invincibles/oom.priority
```

To add more processes to this group, add the pid of the task to the list of tasks in the invincible group: 

```
# echo <pid> > /mnt/oom-killer/invincibles/tasks
```

Important processes, such as database processes and their controllers, can be added to this group, so they are ignored when OOM-killer searches for processes to be killed. All children of the processes listed in tasks automatically are added to the same control group and inherit the `oom.priority` of the parent. When multiple tasks have the highest `oom.priority`, the OOM killer selects the process based on the `oom_score` and `oom_adj`. 

This approach did not appeal to cpuset users, though. Consider two cpusets, A and B. If a process in cpuset A has a high `oom.priority` value, it will be killed if cpuset B runs out of memory, even though there is enough memory in cpuset A. This calls for a different design to tame the OOM killer. 

An interesting outcome of the discussion has been handling OOM situations in user space. The kernel sends notification to user space, and applications respond by dropping their user-space caches. In case the user-space processes are not able to free enough memory, or the processes ignore the kernel's requests to free memory, the kernel resorts to the good old method of killing processes. [mem_notify](<http://lwn.net/Articles/267013/>), developed by Kosaki Motohiro, is one such attempt made in the past. However, the [mem_notify patch](<http://lwn.net/Articles/268732/>) cannot be applied to versions beyond 2.6.28 because the memory management reclaiming sequence have changed, but the design principles and goals can be reused. David Rientjes [suggests](<http://lkml.org/lkml/2009/1/27/49>) having one of the two hybrid solutions: 

One is the cgroup OOM notifier that allows you to attach a task to wait on an OOM condition for a collection of tasks. This allows userspace to respond to the condition by dropping caches, adding nodes to a cpuset, elevating memory controller limits, sending a signal, etc. It can also defer to the kernel OOM killer as a last resort. 

The other is /dev/mem_notify that allows you to poll() on a device file and be informed of low memory events. This can include the cgroup oom notifier behavior when a collection of tasks is completely out of memory, but can also warn when such a condition may be imminent. I suggested that this be implemented as a client of cgroups so that different handlers can be responsible for different aggregates of tasks. 

Most developers prefer making `/dev/mem_notify` a client of control groups. This can be further extended to merge with the proposed oom-controller. 

### Low Memory in Embedded Systems

The Android developers required a greater degree of control over the low memory situation because the OOM killer does not kick in till late in the low memory situation, i.e. till all the cache is emptied. Android wanted a solution which would start early while the free memory is being depleted. So they introduced the "lowmemory" driver, which has multiple thresholds of low memory. In a low-memory situation, when the first thresholds are met, background processes are notified of the problem. They do not exit, but, instead, save their state. This affects the latency when switching applications, because the application has to reload on activation. On further pressure, the lowmemory killer kills the non-critical background processes whose state had been saved in the previous threshold and, finally, the foreground applications. 

Keeping multiple low memory triggers gives the processes enough time to free memory from their caches because in an OOM situation, user-space processes may not be able to run at all. All it takes is a single allocation from the kernel's internal structures, or a page fault to make the system run out of memory. An earlier notification of a low-memory situation could avoid the OOM situation with a little help from the user space applications which respond to low memory notifications. 

Killing processes based on kernel heuristics is not an optimal solution, and these new initiatives of offering better control to the user in selecting the process to be the sacrificial lamb are steps to a robust design to give more control to the user. However, it may take some time to come to a consensus on a final control solution.
