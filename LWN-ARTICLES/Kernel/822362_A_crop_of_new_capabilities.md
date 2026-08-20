---
title: A crop of new capabilities
url: https://lwn.net/Articles/822362/
date: "June 8, 2020"
category: "BPF-Security; Capabilities; Checkpointing"
author: "By Jonathan Corbet June 8, 2020"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
June 8, 2020

Linux capabilities empower the holder to perform a set of specific privileged operations while withholding the full power of root access; see [the capabilities man page](<https://man7.org/linux/man-pages/man7/capabilities.7.html>) for a list of current capabilities and what they control. There have been no capabilities added to the kernel since [`CAP_AUDIT_READ`](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3a101b8de0d3>) was merged for 3.16 in 2014. That's about to change with the 5.8 release, though, which is set to contain two new capabilities; yet another is currently under development. 

#### New capabilities in 5.8

The first of the new capabilities is `CAP_PERFMON`, which was [covered in detail](<https://lwn.net/Articles/812719/>) here last February. With this capability, a user can perform performance monitoring, attach BPF programs to tracepoints, and other related actions. In current kernels, the catch-all `CAP_SYS_ADMIN` capability is required for this sort of performance monitoring; going forward, users can be given more restricted access. Of course, a process with `CAP_SYS_ADMIN` will still be able to do performance monitoring as well; it would be nice to remove that power from `CAP_SYS_ADMIN`, but doing so would likely break existing systems. 

The other new capability, `CAP_BPF`, controls many of the actions that can be carried out with the [`bpf()`](<https://www.man7.org/linux/man-pages/man2/bpf.2.html>) system call. This capability has been the subject of a number of long and intense conversations over the last year; see [this thread](<https://lwn.net/ml/netdev/20190627201923.2589391-1-songliubraving@fb.com/>) or [this one](<https://lwn.net/ml/netdev/20190827205213.456318-1-ast@kernel.org/>) for examples. The original idea was to provide [a special device called `/dev/bpf`](<https://lwn.net/Articles/792124/>) that would control access to [BPF](<https://lwn.net/Articles/740157/>) functionality, but that proposal did not get far. What was being provided was, in essence, a new capability, so capabilities seemed like a better solution. 

The current `CAP_BPF` controls a number of BPF-specific operations, including the creation of BPF maps, use of a number of advanced BPF program features ([bounded loops](<https://lwn.net/Articles/794934/>), cross-program function calls, etc.), access to [BPF type format (BTF)](<https://www.kernel.org/doc/html/latest/bpf/btf.html>) data, and more. While the original plan was to _not_ retain backward compatibility for processes holding `CAP_SYS_ADMIN` in an attempt to avoid what Alexei Starovoitov [described](<https://lwn.net/ml/netdev/20190828044340.zeha3k3cmmxgfqj7@ast-mbp.dhcp.thefacebook.com/>) as the ""deprecated mess"", the code that was actually merged does still recognize `CAP_SYS_ADMIN`. 

One interesting aspect of `CAP_BPF` is that, on its own, it does not confer the power to do much that is useful. Crucially, it is still not possible to load most types of BPF programs with just `CAP_BPF`; to do that, a process must hold other capabilities relevant to the subsystem of interest. For example, programs for tracepoints, kprobes, or perf events can only be loaded if the process also holds `CAP_PERFMON`. Most program types related to networking (packet classifiers, [XDP](<https://lwn.net/Articles/750845/>) programs, etc.) require `CAP_NET_ADMIN`. If a user wants to load a program for a networking function that calls `bpf_trace_printk()`, then both `CAP_NET_ADMIN` and `CAP_PERFMON` are required. It is thus the _combination_ of `CAP_BPF` with other capabilities that grants the ability to use BPF in specific ways. 

Additionally, some BPF operations still require `CAP_SYS_ADMIN`. Offloading BPF programs into hardware is one example. Another one is iterating through BPF objects — programs, maps, etc. — to see what is loaded in the system. The ability to look up a map, for example, would give a process the ability to change maps belonging to other users and with it, the potential for all sorts of mayhem. Thus the bar for such activities is higher. 

The end result of this work is that it will be possible to do quite a bit of network administration, performance monitoring, and tracing work without full root (or even full `CAP_SYS_ADMIN`) access. 

#### CAP_RESTORE

The [`CAP_RESTORE` proposal](<https://lwn.net/ml/linux-kernel/20200522055350.806609-1-areber@redhat.com/>) was posted in late May; its purpose is to allow the checkpointing and restoring of processes by (otherwise) unprivileged processes. Patch author Adrian Reber wrote that this is nearly possible today using the [checkpoint/restore in user space](<https://lwn.net/Articles/525675/>) (CRIU) feature that has been under development for many years. There are a few remaining obstacles, though, one of which is process IDs. Ideally, a process could be checkpointed and restored, perhaps on a different system, without even noticing that anything had happened. If the process's ID changes, though, that could be surprising and could lead to bad results. So the CRIU developers would like the ability to restore a process using the same ID (or IDs for a multi-threaded process) it had when it was checkpointed, assuming that the desired IDs are available, of course. 

Setting the ID of a new process is possible with [`clone3()`](<https://lwn.net/Articles/792628/>), but this feature is not available to unprivileged processes. The ability to create processes with a chosen ID would make a number of attacks easier, so ID setting is restricted to processes with, of course, `CAP_SYS_ADMIN`. Administrators tend to balk at handing out that capability, so CRIU users have been resorting to a number of workarounds; Reber listed a few that vary from the reasonable to the appalling: 

  * Containers that can be put into user namespaces can, of course, control process IDs within their namespaces without any particular difficulty. But that is evidently not a solution for everybody. 
  * Some high-performance computing users run CRIU by way of a setuid wrapper to gain the needed capabilities. 
  * Some users run [the equivalent of a fork bomb](<https://github.com/twosigma/set_ns_last_pid>), quickly creating (and killing) processes to cycle through the process-ID space up to the desired value. 
  * Java virtual-machine developers would evidently like to use CRIU to short out their long startup times; they have been simply patching out the `CAP_SYS_ADMIN` checks in their kernel (a workaround that led Casey Schaufler to [exclaim](<https://lwn.net/ml/linux-kernel/dc86dffb-c7f8-15bb-db4e-be135da650cc@schaufler-ca.com/>): ""That's not a workaround, it's a policy violation. Bad JVM! No biscuit!""). 

Reber reasonably suggested that it should be possible to find a better solution than those listed above, and said that `CAP_RESTORE` would be a good fit. 

Discussion of this patch focused on a couple of issues, starting with whether it was needed at all. Schaufler, in particular, wanted to know what the new capability would buy, and whether it would truly be sufficient to carry out the checkpoint and restore operations without still needing `CAP_SYS_ADMIN`. Just splitting something out of `CAP_SYS_ADMIN`, he [said](<https://lwn.net/ml/linux-kernel/cc9e5272-795f-9275-ba93-90233bf1addd@schaufler-ca.com/>), is not useful by itself: 

If we broke out CAP_SYS_ADMIN properly we'd have hundreds of capabilities, and no one would be able to manage the capability sets on anything. Just breaking out of CAP_SYS_ADMIN, especially if the process is going to need other capabilities anyway, gains you nothing. 

It does seem that `CAP_RESTORE` may, in the end, be sufficient for this task, though, so Schaufler's objections seemed to fade over time. 

The other question that came up was: what other actions would eventually be made possible with this new capability? The patch hinted at others, but they were not implemented. The main one appears to be the ability to read the entries in `/proc/_pid_ /map_files` in order to be able to properly dump out various mappings during the checkpoint procedure. The next version of the patch will have an implementation of this behavior as well. Some developers wondered whether there should be _two_ new capabilities, with the second being `CAP_CHECKPOINT`, to cover the actions specific to each procedure; that change may not happen without further pressure, though. 

The final form of this patch remains to be seen; security-related changes can require a lot of discussion and modification before they find their way in. But this capability seems useful enough that it will probably end up merged in some form at some point.
