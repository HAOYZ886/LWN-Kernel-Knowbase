---
title: "Custom out-of-memory killers in BPF"
url: https://lwn.net/Articles/1019230/
date: "May 1, 2025"
category: "BPF-Memory management; Memory management-Out-of-memory handling"
author: "By Jonathan Corbet May 1, 2025"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
May 1, 2025

The out-of-memory (OOM) killer has long been a scary and controversial part of the Linux kernel. It is summoned from some dark place when the system as a whole (or, more recently, any given control group) is running so low on memory that further allocations are not possible; its job is to kill off processes until a sufficient amount of memory has been freed. Roman Gushchin has found a way to make the OOM killer even scarier: adding the ability to [load custom OOM killers in BPF](<https://lwn.net/ml/all/20250428033617.3797686-1-roman.gushchin@linux.dev>). 

The kernel, in its default configuration, will overcommit the memory available on the system; it will allow processes to allocate more memory than can be provided (that is, more than the sum of physical memory and swap space). Applications routinely allocate more memory than they use; limiting allocations to the available memory would, as a result, cause some of that memory to be unused. Overcommitting memory avoids that waste, and it _almost_ always works out in the end. 

The rare occasion where it doesn't work out is reminiscent of the (not even remotely rare enough) situation where an airline overbooks the seats on a flight. When too many passengers actually show up, some unlucky person will lose their seat. In the Linux world, instead, some process loses its memory — and, with it, the ability to run at all. The kernel is not able to broadcast a request for volunteers to be killed, though, so the OOM killer has to apply a set of heuristics in an attempt to find the victim that will free the most memory while minimizing user anguish. 

Unsurprisingly, the choices made by the OOM killer often align poorly with the decisions the human(s) using the computer would have made. The kernel provides a set of knobs to help tune the heuristics, essentially by allowing some processes to volunteer (or be volunteered) to be the first OOM-killer targets. In recent years, as well, there has been a lot of effort that has gone into user-space OOM killers; these include tools like [oomd](<https://facebookmicrosites.github.io/oomd/>), [systemd-oomd](<https://www.freedesktop.org/software/systemd/man/latest/systemd-oomd.service.html>), and the [Android lmkd](<https://source.android.com/docs/core/perf/lmkd>). 

User-space OOM killers have some fundamental problems, though. Since they run in user space, they will necessarily be slower than the kernel to respond to a low-memory situation, which is a problem that urgently requires a solution. Running the user-space OOM killer may, itself, require allocating memory at the worst possible time. User-space killers may also lack useful information about the state of the system that may be available within the kernel. While OOM killers running in user space can make use of information about the specific workload running on the system, leading to better decisions, they can fall short in other ways. 

Gushchin's series aims to address these problems by providing two hooks for BPF programs to implement the OOM-handling task, enabling OOM killers that benefit from user-space control while running inside the kernel. The first of these hooks (even though it comes second in the series) is a hook that is called in response to events from the [pressure stall information (PSI) subsystem](<https://docs.kernel.org/accounting/psi.html>): 

```
int bpf_handle_psi_event(struct psi_trigger *t);
```

A program that attaches to this hook will be invoked whenever a PSI event triggers, indicating that a threshold has been exceeded and that memory pressure is causing the workload to slow down. That program can use the PSI information, along with anything else it might have at hand, to come to its own conclusion about the state of the system. Should the judgment be that the time has come to push the panic button, this program can declare an OOM event with a new kfunc: 

```
int bpf_out_of_memory(struct mem_cgroup *memcg, int order);
```

The `memcg` parameter, if non-NULL, limits the OOM event to the given memory control group; `order` describes the order of the allocation being attempted — which is not entirely applicable in this situation, since memory is not actually being allocated. This call will summon the OOM killer to deal with the problem. 

It is worth noting that the kernel is only able to handle one OOM event at a time; if one control group is on fire, the kernel cannot respond to OOM situations in any others. In that case, `bpf_out_of_memory()` will return `-EBUSY`. 

The other new hook comes into play once the OOM apocalypse has struck: 

```
int bpf_handle_out_of_memory(struct oom_control *oc);
```

This function, which will be called to answer the invocation of the OOM killer, should do something to address the OOM situation, returning a non-zero value if it succeeds in freeing some memory. The kernel checks after the BPF program runs to be sure that memory was _really_ freed; it does not just take the program's word for it. If the BPF program is unable to free memory, the normal kernel OOM killer will step in to try to get the job done properly. 

Since "do something" likely involves killing processes, the series provides a handy new kfunc to do just that: 

```
int bpf_oom_kill_process(struct oom_control *oc, struct task_struct *task,
    			     const char *message);
```

This function will bring about the untimely demise of the process indicated by `task`, updating `oc` to indicate whether memory was successfully freed. It is worth noting that `bpf_oom_kill_process()` is made available to all BPF programs of the tracing type, which normally are not in the business of killing processes. There is another new kfunc, `bpf_get_root_mem_cgroup()`, that can be used to get the root of the control-group hierarchy. The OOM-killer program can then traverse that hierarchy in search of the best victim. 

Of course, the OOM-killer program could also take other actions to address the memory problem. Gushchin suggests deleting tmpfs files as one example. Or, depending on the application being run, the program could request that memory be freed in some other way that does not involve killing off processes. 

This is an RFC patch set that is not intended to be applied in its current form. As of this writing, there have been few comments. Michal Hocko [suggested](<https://lwn.net/ml/all/aBC7E487qDSDTdBH@tiehlicka>) that it might make sense splitting `bpf_handle_out_of_memory()` to handle the different cases (global, control group, or CPUset) cases separately; he also asked whether real OOM handlers have been implemented with this infrastructure. 

It seems likely that the prospect of giving BPF programs the ability to run around and kill processes will make some people uncomfortable. Of course, the OOM killer _always_ makes people uncomfortable. But, as long as this creature must exist, it may be best if it runs within the kernel.
