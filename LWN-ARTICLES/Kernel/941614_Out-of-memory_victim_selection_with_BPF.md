---
title: "Out-of-memory victim selection with BPF"
url: https://lwn.net/Articles/941614/
date: "August 17, 2023"
category: "BPF-Memory management; Memory management-Out-of-memory handling"
author: "By Jonathan Corbet August 17, 2023"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
August 17, 2023

In its default configuration, the Linux kernel will allow processes to allocate more memory than the system can actually provide; this policy enables better utilization of physical memory and works just fine — most of the time. On occasions, though, the kernel may find itself unable to provide memory that processes may think already belongs to them. If the situation gets bad enough, the only solution (short of rebooting) is to declare a sort of memory bankruptcy and write off some of the kernel's debts by killing one or more processes. Over the years, a great deal of effort has gone into heuristics to select the processes that the user is least likely to miss. This problem is still clearly not solved to everybody's satisfaction, though, so it was only a matter of time before somebody introduced a way to select the out-of-memory (OOM) victim using BPF. 

There are numerous ways to go hunting for a process to sacrifice when memory runs out. The process using the most memory is an obvious choice, but that process is often something important: a window-system server or a database manager, for example. So developers have naturally tried, over the years, to enable the kernel to make a better choice; see [the LWN kernel index](<https://lwn.net/Kernel/Index/#Memory_management-Out-of-memory_handling>) to see how things have evolved over time. In current kernels, this decision comes down to a function called [`oom_badness()`](<https://elixir.bootlin.com/linux/v6.4.10/source/mm/oom_kill.c#L191>) which, after exempting processes that cannot be killed for one reason or another, makes a simple calculation. A process's "OOM score" comes down to the amount of memory it uses, adjusted by that process's [`oom_score_adj` value](<https://www.kernel.org/doc/html/latest/filesystems/proc.html#chapter-3-per-process-parameters>). By tweaking those knobs, user space can shelter some processes from the OOM-killer's depredations while directing its attention toward others. 

That, evidently, is not enough control for some users. The [BPF patch series](<https://lwn.net/ml/linux-kernel/20230810081319.65668-1-zhouchuyi@bytedance.com/>) from Chuyi Zhou is the latest in a series of attempts to improve that control. 

In current kernels, the OOM killer will iterate through all of the possible target processes, call `oom_badness()` on each, then target the process that is given the highest score. Zhou's patch set allows the `oom_badness()` check to be replaced with a call to a BPF function, which should be defined as an "fmod_ret" tracing function (meaning it is invoked on return from an internal kernel function and can change that function's return value) with this name and prototype: 

```
int bpf_oom_evaluate_task(struct task_struct *task, struct oom_control *oc);
```

This function will be called at the beginning of the evaluation of each potential victim and, if it makes a decision on the given `task`, will cause the normal evaluation to be skipped. The [`oom_control` structure](<https://elixir.bootlin.com/linux/v6.4.10/source/include/linux/oom.h#L29>) describes the context in which the OOM kill is taking place; the BPF function has access to it but probably (the rules are not actually documented anywhere) should not make changes to it. That function can also look at the `task` under consideration and make a decision regarding its fate, as reflected in its return value: 

  * `NO_BPF_POLICY`: no policy is in effect, so the normal `oom_badness()` method should be used. 
  * `BPF_EVAL_ABORT`: abort the selection process entirely with no process chosen to kill. 
  * `BPF_EVAL_NEXT`: move on to the next process, passing over this one. 
  * `BPF_EVAL_SELECT`: select this process as the one to kill. 

Returning `BPF_EVAL_SELECT` does not bring an end to the iteration through the list of processes; there will be further calls to `bpf_oom_evaluate_task()` if there are more processes to examine. As a result, the function can change its mind and return `BPF_EVAL_SELECT` again if a more appealing victim comes along later in the sequence. 

It is possible to use `BPF_EVAL_NEXT` for some processes while using `NO_BPF_POLICY` for others. The end result will be to shield some processes from the OOM killer while letting the kernel make a decision in the usual way by looking at the rest. Mixing `BPF_EVAL_SELECT` and `NO_BPF_POLICY` looks like it could create surprising results, though; this combination does not appear to be intended and should, unless something changes in a future version, be avoided. 

Specifically, the `oom_control` structure contains a pointer called `chosen` identifying the currently selected victim, and an integer `chosen_points` holding its badness score. In the absence of a BPF program, the kernel compares each process's score against `chosen_points`, and updates both if the new process has a higher score. Returning `BPF_EVAL_SELECT` sets `chosen` without setting `chosen_points` to anything. If `BPF_NO_POLICY` is returned for a later process, its score will be compared against a `chosen_points` that has no connection to the process selected earlier. 

There are two related hooks provided by the patch set as well. One of them allows the name of the current victim-selection policy to be stored in the kernel; that name will be propagated through to the log when an actual kill is done. To do so, the program should define an fmod_ret function: 

```
void bpf_set_policy_name(struct oom_control *oc);
```

That function, which will be called at the beginning of the OOM-kill procedure, can then turn around and call: 

```
void set_oom_policy_name(struct oom_control *oc, const char *name, size_t sz);
```

Where `oc` is the `oom_control` structure passed to `bpf_set_policy_name()`, `name` is the policy name to use, and `sz` is the length of that name. Names are limited to 16 bytes, including the terminating NUL byte. 

There is also a new tracepoint, `select_bad_process_end`, that fires if the OOM-kill procedure fails to find a process to kill. It is intended to be a helper for developers who are trying to develop a new OOM-kill policy. 

This series is currently in its second revision. In response to [the first posting](<https://lwn.net/ml/linux-kernel/20230804093804.47039-1-zhouchuyi@bytedance.com/>), memory-management developer Michal Hocko [suggested](<https://lwn.net/ml/linux-kernel/ZMzhDFhvol2VQBE4@dhcp22.suse.cz/>) simplifying the interface somewhat. Roman Gushchin, instead, [argued](<https://lwn.net/ml/linux-kernel/ZNEpsUFgKFIAAgrp@P9FQF9L96D.lan/>) for taking a more general approach, where the BPF program is called once at OOM time and is expected to figure out a way to free some memory somewhere. Hocko [responded](<https://lwn.net/ml/linux-kernel/ZNH6X/2ZZ0quKSI6@dhcp22.suse.cz/>) that it would be better to start with ""something that is good enough"" add complexity later if it seems warranted. In response to the second revision, Alexei Starovoitov [also supported](<https://lwn.net/ml/linux-kernel/CAADnVQK=7NWbRtJyRJAqy5JwZHRB7s7hCNeGqixjLa4vB609XQ@mail.gmail.com/>) a more general callback, though, and Zhou has [started considering](<https://lwn.net/ml/linux-kernel/93627e45-dc67-fd31-ef43-a93f580b0d6e@bytedance.com/>) the implications of such a change. 

Both Hocko and Gushchin expressed worries that introducing BPF into this code, which runs when the system is in a distressed state, could further reduce the stability of an out-of-memory situation. An attempt by a BPF program to allocate a lot of memory in this situation seems likely to end in tears, for example. That is true of any code that hooks into the OOM-killer, though, and is not a problem specific to BPF. 

The conversation has shown that there is some interest in the use of BPF to select victims for the out-of-memory killer. Thus far, though, there is not a clear consensus on the approach that this work should take. It would not be surprising, at this point, to see this feature go through some significant changes before it gets closer to the mainline; until then, the kernel will just have to continue choosing processes to sacrifice the old-fashioned way.
