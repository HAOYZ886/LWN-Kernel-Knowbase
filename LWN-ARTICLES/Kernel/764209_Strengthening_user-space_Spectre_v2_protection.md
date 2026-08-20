---
title: "Strengthening user-space Spectre v2 protection"
url: https://lwn.net/Articles/764209/
date: "September 5, 2018"
category: "Security-Meltdown and Spectre"
author: "By Jonathan Corbet September 5, 2018"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
September 5, 2018

The Spectre variant 2 vulnerability allows the speculative execution of incorrect (in an attacker-controllable way) indirect branch predictions, resulting in the ability to exfiltrate information via side channels. The kernel has been reasonably well protected against this variant since shortly after its disclosure in January. It is, however, possible for user-space processes to use Spectre v2 to attack each other; thus far, the mainline kernel has offered relatively little protection against such attacks. A recent [proposal](<https://lwn.net/ml/linux-kernel/nycvar.YFH.7.76.1809041619510.15880@cbobk.fhfr.pm/>) from Jiri Kosina may change that situation, but there are still some disagreements around the details. 

On relatively recent processors (or those with suitably patched microcode), the "indirect branch prediction barrier" (IBPB) operation can be used to flush the branch-prediction buffer, removing any poisoning that an attacker might have put there. Doing an IBPB whenever the kernel switches execution from one process to another would defeat most Spectre v2 attacks, but IBPB is seen as being expensive, so this does not happen. Instead, the kernel looks to see whether the incoming process has marked itself as being non-dumpable, which is typically only done by specialized processes that want to prevent secrets from showing up in core dumps. In such cases, the process is deemed to be worth protecting and the IBPB is performed. 

Kosina notes that only a ""negligible minority"" of the code running on Linux systems marks itself as non-dumpable, so user space on Linux systems is essentially unprotected against Spectre v2. The solution he proposes is to use IBPB more often. In particular, the new code checks whether the outgoing process would be able to call [`ptrace()`](<http://man7.org/linux/man-pages/man2/ptrace.2.html>) on the incoming process. If so, the new process can keep no secrets from the old one in any case, so there is no point in executing an IBPB operation. In cases where `ptrace()` would not succeed, though, the IBPB will happen. 

This code, Kosina said, has been shipping in SUSE kernels since the initial disclosure of the Spectre vulnerabilities. 

Comments on this change focused on two specific areas. One was the fact that Casey Schaufler has been working on a ["sidechannel" Linux security module](<https://lwn.net/Articles/763058/>) (LSM) specifically intended to make decisions on when operations like IBPB should be performed. This module leaves it up to the security policy to decide when the possibility of an attack exists; different security mechanisms might make different decisions here. Kosina's patch, instead, wires the policy directly into the kernel. Schaufler [argued](<https://lwn.net/ml/linux-kernel/99FC4B6EFCEFD44486C35F4C281DC67321447094@ORSMSX107.amr.corp.intel.com/>) in this discussion that using `ptrace()` was the wrong approach: ""Even if ptrace_may_access() does exactly what you want it to for side-channel mitigation today it would be incredibly inappropriate to tie the two together for eternity."" 

Kosina initially [agreed](<https://lwn.net/ml/linux-kernel/nycvar.YFH.7.76.1809042046180.15880@cbobk.fhfr.pm/>) to drop his patches in favor of the sidechannel module, but later backtracked for a couple of reasons, including the fact that this module is not ready for production use yet. Beyond that, giving the administrator the flexibility to choose among policies has its advantages, but those advantages come at a cost, he [said](<https://lwn.net/ml/linux-kernel/nycvar.YFH.7.76.1809050725390.15880@cbobk.fhfr.pm/>): 

I am a bit afraid that we are offloading to sysadmins decisions that are very hard for them to make, as they require deep understanding of both the technical details of the security issue in the CPU, and the mitigation. 

Instead, he said, the only choice provided should be whether protection against Spectre v2 is needed or not. 

There are also questions about whether security modules are the right place to make decisions on the use of mitigations for hardware vulnerabilities. Andrea Arcangeli [noted](<https://lwn.net/ml/linux-kernel/20180905023832.GM4762@redhat.com/>) that the other defenses against Meltdown and Spectre are not controlled by security modules: 

Even if you build with CONFIG_SECURITY=n PTI won't go away, retpoline won't go away, the l1flush in vmenter won't go away, the pte/transhugepmd inversion won't go away, why only the runtime tunability or tweaking of IBPB fits in a LSM module? 

For now, Kosina plans to continue to push the patch set using the `ptrace()` check for now; ""we can then later see whether the LSM implementation, once it exists, should be used instead"". 

The other question on developers' minds was the performance impact of this change; Kosina did not include any numbers with the patch set. Andi Kleen [complained](<https://lwn.net/ml/linux-kernel/20180905155823.GL27886@tassilo.jf.intel.com/>) about that omission, saying ""It's ridiculous to even discuss this without them"". Arcangeli [asserted](<https://lwn.net/ml/linux-kernel/20180904233714.GJ4762@redhat.com/>) that ""IBPB has never been measurable if done only when the prev task cannot ptrace the next task"", but he, too, did not offer numbers to back that claim up. It seems certain that some real measurements will be required before this code can go upstream. On the other hand, some developers clearly see the security aspect as being the most important; as Thomas Gleixner [put it](<https://lwn.net/ml/linux-kernel/alpine.DEB.2.21.1809052025250.1416@nanos.tec.linutronix.de/>): ""Either we care about that problem and provide a proper mechanism to protect systems or we do not. That's not a performance number problem at all"". 

One other aspect of the Spectre problem is, inevitably, hyperthreading. Sibling CPUs share resources, including the branch-prediction buffer, so it may be possible for code running on one sibling to attack the process running on the other. The "single thread indirect branch predictors" (STIBP) feature provided on some Intel CPUs disables this sharing, and can thus block those attacks. The kernel does not currently use STIBP, but Kosina's patch set adds it, leaving it enabled whenever hyperthreading is in use. Again, no numbers showing the performance impact of the change were provided. 

Once the dust settles, the kernel is likely to end up with increased protection against Spectre v2 for user-space processes. It might seem unfortunate that this fix is not likely to find its way into a released kernel until a full year after the disclosure of this vulnerability; Kosina clearly [thinks so](<https://lwn.net/ml/linux-kernel/nycvar.YFH.7.76.1809052029160.15880@cbobk.fhfr.pm/>). On the other hand, it is not clear that there are a lot of attackers out there trying to use this vulnerability against user-space processes. Dealing with hardware always involves tradeoffs and workarounds for difficult behavior; the best ways to handle such behavior can take some time to work out. The kernel developers will eventually settle on the best ways to deal with Spectre v2. That would be more comforting, of course, if we had any confidence that the flow of speculative-execution vulnerabilities will slow down sometime soon.
