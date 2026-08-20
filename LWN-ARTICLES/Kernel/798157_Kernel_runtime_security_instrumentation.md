---
title: Kernel runtime security instrumentation
url: https://lwn.net/Articles/798157/
date: "September 4, 2019"
category: "BPF-Security; Security-Security modules"
author: "By Jake Edge September 4, 2019 LSS-NA"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
September 4, 2019

* * *

[LSS-NA](<https://lwn.net/Archives/ConferenceByYear/#2019-Linux_Security_Summit_North_America>)

Finding ways to make it easier and faster to mitigate an ongoing attack against a Linux system at runtime is part of the motivation behind the kernel runtime security instrumentation (KRSI) project. Its developer, KP Singh, gave a presentation about the project at the [2019 Linux Security Summit North America](<https://events.linuxfoundation.org/events/linux-security-summit-north-america-2019/>) (LSS-NA), which was held in late August in San Diego. A prototype of KRSI is implemented as a Linux security module (LSM) that allows eBPF programs to be attached to the kernel's security hooks. 

Singh began by laying out the motivation for KRSI. When looking at the security of a system, there are two sides to the coin: signals and mitigations. The signals are events that might, but do not always, indicate some kind of malicious activity is taking place; the mitigations are what is done to thwart the malicious activity once it has been detected. The two "go hand in hand", he said. 

For example, the audit subsystem can provide signals of activity that might be malicious. If you have a program that determines that the activity actually is problematic, then you might want it to update the policy for an LSM to restrict or prevent that behavior. Audit may also need to be configured to log the events in question. He would like to see a unified mechanism for specifying both the signals and mitigations so that the two work better together. That is what KRSI is meant to provide. 

He gave a few examples of different types of signals. For one, a process that executes and then deletes its executable might well be malicious. A kernel module that loads and then hides itself is also suspect. A process that executes with suspicious environment variables (e.g. `LD_PRELOAD`) might indicate something has gone awry as well. 

On the mitigation side, an administrator might want to prevent mounting USB drives on a server, perhaps after a certain point during the startup. There could be dynamic whitelists or blacklists of various sorts, for kernel modules that can be loaded, for instance, to prevent known vulnerable binaries from executing, or stopping binaries from loading a core library that is vulnerable to ensure that updates are done. Adding any of these signals or mitigations requires reconfiguration of various parts of the kernel, which takes time and/or operator intervention. He wondered if there was a way to make it easy to add them in a unified way. 

#### eBPF + LSM

He has created a new eBPF program type that can be used by the KRSI LSM. There is a set of eBPF helpers that provide a "unified policy API" for signals and mitigations. They are security-focused helpers that can be built up to create the behavior required. 

[ ![\[KP Singh\]](https://static.lwn.net/images/2019/lssna-singh-sm.jpg) ](<https://lwn.net/Articles/798172/>)

Singh is frequently asked why he chose to use an LSM, rather than other options. Security behaviors map better to LSMs, he said, than to things like seccomp filters, which are based on system call interception. Various security-relevant behaviors can be accomplished via multiple system calls, so it would be easy to miss one or more, whereas the LSM hooks intercept the behaviors of interest. He also hopes this work will benefit the overall LSM ecosystem, he said. 

He talked with some security engineers about their needs and one mentioned logging `LD_PRELOAD` values on process execution. The way that could be done with KRSI would be to add a BPF program to to the [`bprm_check_security()`](<https://elixir.bootlin.com/linux/v5.2.11/source/include/linux/lsm_hooks.h#L51>) LSM hook that gets executed when a process is run. So KRSI registers a function for that hook, which gets called along with any other LSM's hooks for `bprm_check_security()`. When the KRSI hook is run, it calls out to the BPF program, which will communicate to user space (e.g. a daemon that makes decisions to add further restrictions) via an output buffer. 

The intent is that the helpers are "precise and granular". Unlike the BPF tracing API, they will not have general access to internal kernel data structures. His [slides [PDF]](<https://static.sched.com/hosted_files/lssna19/03/Kernel%20Runtime%20Security%20Instrumentation.pdf>) had [`bpf_probe_read()`](<https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md#1-bpf_probe_read>) in a circle with a slash through it as an indication of what he was trying to avoid. The idea is to maintain backward compatibility by not tying the helpers to the internals of a given kernel. 

He then went through various alternatives for implementing this scheme and described the problems he saw with them. To start with, why not use audit? One problem is that the mitigations have to be handled separately. But there is also a fair amount of performance overhead when adding more things to be audited; he would back that up with some numbers later in the presentation. Also, audit messages have rigid formatting that must be parsed, which might delay how quickly a daemon could react. 

Seccomp with BPF was up next. As he said earlier, security behaviors map more directly into LSM hooks than to system-call interception. He is also concerned about time-of-check-to-time-of-use (TOCTTOU) races when handling the system call parameters from user space, though he said he is not sure that problem actually exists. 

Using kernel probes (kprobes) and eBPF was another possibility. It is a "very flexible" solution, but it depends on the layout of internal kernel data structures. That makes deployment hard as things need to be recompiled for each kernel that is targeted. In addition, kprobes is not a stable API; functions can be added and removed from the kernel, which may necessitate changes. 

The final alternative was the [Landlock LSM](<https://lwn.net/Articles/703876/>). It is geared toward providing a security sandbox for unprivileged processes, Singh said. KRSI, on the other hand, is focused on detecting and reacting to security-relevant behaviors. While Landlock is meant to be used by unprivileged processes, KRSI requires `CAP_SYS_ADMIN` to do its job. 

#### Case study

He then described a case study: auditing the environment variables set when executing programs on a system. It sounds like something that should be easy to do, but it turns out not to be. For one thing, there can be up to 32 pages of environment variables, which he found surprising. 

He looked at two different designs for an eBPF helper, one that would return all of the environment variables or one that just returned the variable of interest. The latter has less overhead, so it might be better, especially if there is a small set of variables to be audited. But either of those helpers could end up sleeping because of a page fault, which is something that eBPF programs are not allowed to do. 

Singh did some rough performance testing in order to ensure that KRSI was not completely unworkable, but the actual numbers need to be taken with a few grains of salt, he said. He ran a no-op binary 100 times and compared the average execution time (over N iterations of the test) of that on a few different systems: a kernel with audit configured out, a kernel with audit but no audit rules, one where audit was used to record `execve()` calls, and one where KRSI recorded the value of `LD_PRELOAD`. The first two were measured at a bit over 500µs (518 and 522), while the audit test with rules came in at 663µs (with a much wider distribution of values than any of the other tests). The rudimentary KRSI test clocked in at 543µs, which gave him reason to continue on; had it been a lot higher, he would have shelved the whole idea. 

There are plenty of things that are up for discussion, he said. Right now, KRSI uses the perf ring buffer to communicate with user space; it is fast and eBPF already has a helper to access it. But that ring buffer is a per-CPU buffer, so it uses more memory than required, especially for systems with a lot of CPUs. There is already talk of allowing eBPF programs to sleep, which would simplify KRSI and allow it to use less memory. Right now, the LSM hook needs to pin the memory for use by the eBPF program. He is hopeful that discussions in the [BPF microconference](<https://linuxplumbersconf.org/event/4/sessions/62/#20190911>) at the [Linux Plumbers Conference](<https://linuxplumbersconf.org>) will make some progress on that. 

As part of the Q&A, Landlock developer Mickaël Salaün spoke up to suggest working together. He went through the same thinking about alternative kernel facilities that Singh presented and believes that Landlock would integrate well with KRSI. Singh said that he was not fully up-to-speed on Landlock but was amenable to joining forces if the two are headed toward the same goals. 

[I would like to thank LWN's travel sponsor, the Linux Foundation, for funding to travel to San Diego for LSS-NA.]
