---
title: The sidechannel LSM
url: https://lwn.net/Articles/763058/
date: "August 21, 2018"
category: "Security-Meltdown and Spectre; Security-Security modules"
author: "By Jake Edge August 21, 2018"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jake Edge**  
August 21, 2018

Side-channel attacks are a reasonably well-known technique to exfiltrate information across security boundaries. Until relatively recently, concerns about these types of attacks were mostly confined to cryptographic operations, where the target was to extract secrets by observing some side channel. But with the advent of Spectre, speculative execution provides a new way to exploit side channels. A new Linux Security Module (LSM) is meant to help determine where a side channel might provide secrets to an attacker, so that a speculative-execution barrier operation can be performed. 

In current kernels, a context switch from one process to another often necessitates a flush of the translation lookaside buffer (TLB) contents, which is done in [`switch_mm_irqs_off()`](<https://elixir.bootlin.com/linux/v4.18.3/source/arch/x86/mm/tlb.c#L183>). For x86, after the Spectre v2 mitigations, that function calls [`indirect_branch_prediction_barrier()`](<https://elixir.bootlin.com/linux/v4.18.3/source/arch/x86/include/asm/nospec-branch.h#L263>) when switching away from a process that is not allowed to core dump (i.e. does not have `SUID_DUMP_USER` set). The barrier (which is known as IBPB) is an expensive operation, so it is only done for "sensitive" processes that have turned off core dumps (e.g. GPG). Core dumps of a process can contain secrets of various sorts, such as keys or passwords. 

However, there may be other sensitive processes that do not turn off core dumps but are still susceptible to this side channel, so a [patch set](<https://lwn.net/ml/kernel-hardening/20180815235355.14908-1-casey.schaufler@intel.com/>) from Casey Schaufler would allow LSMs to offer an opinion on whether the IBPB should be done. It adds a new LSM hook (`task_safe_sidechannel()`) that will return zero if there are no known side-channel worries or `-EACCES` if the LSM considers the context switch to be potentially sensitive. The patch set provides an LSM to check some security attributes of tasks and also adds checking to the SELinux and Smack LSMs so that they can report whether the security attributes they maintain indicate a potential side-channel concern. 

The [SELinux](<https://lwn.net/ml/kernel-hardening/20180815235355.14908-6-casey.schaufler@intel.com/>) and [Smack](<https://lwn.net/ml/kernel-hardening/20180815235355.14908-5-casey.schaufler@intel.com/>) changes add an entry for the new hook. Each looks at the current task and the task to be switched to and renders a verdict on the side-channel safety of the switch. The SELinux hook considers the switch to be safe against side channels if the current task has `FILE__READ` access to the new task. For Smack, it is similar: ""Smack considers its private task data safe if the current task has read access to the passed task."" 

The bulk of the patch set, though, is the [new "sidechannel" LSM](<https://lwn.net/ml/kernel-hardening/20180815235355.14908-4-casey.schaufler@intel.com/>). It is enabled with the `SECURITY_SIDECHANNEL` kernel configuration option, but requires other options in order to actually do any checking. One of them assumes that all task switches are subject to side channels (`SECURITY_SIDECHANNEL_ALWAYS`), so it simply always returns `-EACCES`. The other three enable various checks: 

  * `SECURITY_SIDECHANNEL_UIDS`: checks if the tasks have different effective UIDs and reports side-channel susceptibility if so; this could have a high performance impact since most context switches are between tasks with different effective UIDs. 
  * `SECURITY_SIDECHANNEL_CAPABILITIES`: checks if the tasks have different sets of capabilities, which may mean the new task would be subject to side-channel attacks. 
  * `SECURITY_SIDECHANNEL_NAMESPACES`: checks if the tasks live in different user, PID, or control-group namespaces and returns `-EACCES` if so. 
Given that a distribution will have to enable the LSM to make it available to its users, it would seem to some kind of runtime or load-time configuration of the different levels might be useful. As it stands, the LSM looks like it will only be used by those who build their own kernels. 

The comments on the patch set have been relatively light. Jann Horn has made several suggestions, most of which Schaufler has adopted; the patch set is now up to [v3](<https://lwn.net/ml/kernel-hardening/20180821000444.7004-1-casey.schaufler@intel.com/>). One comment that has not been addressed in the patch set is Horn's [request](<https://lwn.net/ml/kernel-hardening/CAG48ez0=RSok9mAo9QHsho+Qnm_ZDqkH68Amfzj9A_jWmOZaOA@mail.gmail.com/>) that the security checks look at the previous non-kernel task when switching away from the kernel. He [went into more detail](<https://lwn.net/ml/kernel-hardening/CAG48ez1LHOHvB4ud+8asOjKARVLQJGV4ocdKVDJtvTXTfeMa9w@mail.gmail.com/>) in a posting on v2 of the patch set: 

That means that an attacker who can e.g. get a CPU to first switch from an attacker task to a softirqd (e.g. for network packet processing or whatever), then switch from the softirqd to a root-owned victim task would be able to bypass the check, right? That doesn't sound like a very complicated attack... 

I very much dislike the idea of adding a mitigation with a known bypass technique to the kernel. 

The test in `switch_mm_irqs_off()` to decide whether to do the IBPB looks at the task structure; if it is a kernel thread, thus does not have an `mm` pointer to a process address space, the rest of the checks are shorted out. Schaufler didn't change that, though he did "touch" it by adding the new LSM hook call, so Horn's complaint is really about the existing test. Horn suggested keeping a copy of the metadata for the most recent non-kernel task in order to do that test, but Schaufler has not made that change; his [argument](<https://lwn.net/ml/kernel-hardening/99FC4B6EFCEFD44486C35F4C281DC6732143F769@ORSMSX107.amr.corp.intel.com/>) was that those who are concerned about that kind of attack should probably simply enable the "always" option. 

Schaufler was also concerned with finding a good mechanism to save the task metadata. Horn [offered some suggestions](<https://lwn.net/ml/kernel-hardening/CAG48ez0j4+4iaH60Xf857Oh8TwhXzA39M3-CC4kXJKf6Ctvchg@mail.gmail.com/>), but noted that the obvious way to do so might not be favored in a hot path like context switching: ""The obvious solution would be to take a refcounted reference on the old task's objective creds, but you probably want to avoid the resulting cache line bouncing..."" 

It certainly seems reasonable for the LSMs to get involved in the decision on whether a process might be susceptible to a side-channel attack from another process. The current "dumpable" test is a simple one, but likely ignores many sensitive processes. But context switching is an important function of the kernel and one that should be done as quickly as possible. Adding complexity there may not be particularly welcome, but there have been no complaints so far. Speculative execution is done as a performance optimization but clearly we are having to give some of that improvement back to work around the shortcomings of its implementation in some CPUs.
