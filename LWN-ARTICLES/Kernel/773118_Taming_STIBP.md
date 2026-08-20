---
title: Taming STIBP
url: https://lwn.net/Articles/773118/
date: "November 29, 2018"
category: "Security-Meltdown and Spectre"
author: "By Jonathan Corbet November 29, 2018"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
November 29, 2018

The Spectre class of hardware vulnerabilities was apparently so-named because it can be expected to haunt us for some time. One aspect of that haunting can be seen in the fact that, nearly one year after Spectre was disclosed, the kernel is still unable to prevent one user-space process from attacking another in some situations. An attempt to provide that protection using a new x86 microcode feature called `STIBP` has run into trouble once its performance impact was understood; now a more nuanced approach may succeed in providing protection where it is needed without slowing down everybody else. 

The Spectre variant 2 vulnerability works by polluting the CPU's branch-prediction buffer (BPB), which is used during speculative execution to make a guess about which branch(es) the code will take; see [this article](<https://lwn.net/Articles/743265/>) for a refresher on the Spectre vulnerabilities if needed. Closing this hole requires changes at a number of levels, but a fundamental part of the problem is preventing any code that may be targeted from running with a BPB that has been trained by an attacker. 

There are a few ways in which this can be accomplished; in many cases the appropriate tool is a new instruction called `IBPB`, which flushes the BPB. Developers have been [discussing](<https://lwn.net/Articles/764209/>) the right times to execute `IBPB` instructions for some time, but the overall strategy is relatively straightforward: an `IBPB` instruction should be run whenever the CPU switches between tasks that do not trust each other. A few modes for determining when `IBPB` should be used have been implemented and can be selected with command-line options. 

`IBPB` leaves one part of the problem unsolved, though. When simultaneous multithreading (SMT, or "hyperthreading") is in use, two threads of execution are, for all practical purposes, executing on the same CPU simultaneously. Those threads will share the same BPB; if one thread populates the BPB with hostile entries, the other thread will be affected by them until the next `IBPB` instruction is executed. In other words, SMT processors create an ongoing series of time windows in which one thread may attack another, even when `IBPB` is in use. Some security-sensitive users have disabled SMT entirely in response to this problem (and others), but not everybody wants to pay that cost. 

That is where `STIBP` comes in. It is a processor mode (rather than an instruction) that, according to [Intel's press materials [PDF]](<https://newsroom.intel.com/wp-content/uploads/sites/11/2018/01/Intel-Analysis-of-Speculative-Execution-Side-Channels.pdf>), ""prevents indirect branch predictions from being controlled by the sibling Hyperthread"". This sounds like just what is needed to keep threads from attacking each other. After some discussion, [`STIBP` support](<https://git.kernel.org/linus/53c613fe6349994f023245519265999eed75957f>) was added to the kernel during the 4.20 merge window. At that time, the decision was made to enable `STIBP` by default and to leave it on, so that systems would automatically be protected. This patch was subsequently backported to the 4.19.2, 4.18.19, 4.14.81, and 4.9.137 stable updates. 

It turns out, however, that there is a problem with `STIBP`: it slows the system down significantly for many workloads. Linus Torvalds managed to keep his promise to be more polite when he [described what is going on](<https://lwn.net/ml/linux-kernel/CAHk-=whtiX45YPjFPMFuktZ3WB23zqBZR-rL6RewrTyvrFj2Fw@mail.gmail.com/>), but it must have been a strain: 

Yes, Intel calls it "STIBP" and tries to make it out to be about the indirect branch predictor being per-SMT thread. 

But the reason it is unacceptable is apparently because in reality it just disables indirect branch prediction entirely. So yes, *technically* it's true that that limits indirect branch prediction to just a single SMT core, but in reality it is just a "go really slow" mode. 

As reports of performance regressions started rolling in, it became clear that the decision to enable `STIBP` by default would have to be revisited. In the resulting [discussion](<https://lwn.net/ml/linux-kernel/CAHk-=wg-9FUGU=grF4gKDq1sm1m39Jbs3A_iyLbSSntU47ncwg@mail.gmail.com/>), Torvalds said that `STIBP` needed to be made an optional feature that could be enabled by ""crazy people"" who are willing to pay the performance cost it brings. Arjan van de Ven [said](<https://lwn.net/ml/linux-kernel/51127fd4-5dcc-b2b9-4873-72098d2a77d9@linux.intel.com/>) that both Intel and AMD recommend against enabling it by default (though Intel has apparently not actually documented that recommendation anywhere). Ingo Molnar [promised](<https://lwn.net/ml/linux-kernel/20181119083855.GA129733@gmail.com/>) to require performance measurements for any future mitigations before they can be merged. The `STIBP` patch was reverted in the 4.19.4 4.14.83, and 4.9.140 stable updates; it remains in 4.18 since that series is no longer receiving updates. 

As of this writing, the `STIBP` patch is also still in the mainline kernel, pending the finalization of a better solution. That solution is likely to take the form of [this patch set](<https://lwn.net/ml/linux-kernel/20181125183328.318175777@linutronix.de/>) posted by Thomas Gleixner, containing the work of a number of developers. `STIBP` is disabled on any system that does not actually have running processors with SMT enabled, even if such processors could materialize in the future. It is also disabled by default for most processes on the system, but it can be globally enabled with the `spectre_v2_user=on` command-line option. 

There is also a new set of values for the `spectre_v2=` command-line option that can be used to enable more control over branch prediction: 

  * `spectre_v2=prctl` leaves both `IBPB` and `STIBP` disabled by default, but allows them to be enabled for individual processes via a new `prctl()` operation. In this mode, the system can generally run without the extra overhead of the Spectre mitigations, but those mitigations can be turned on for specific processes that need extra protection. 
  * `spectre_v2=seccomp` is the same as the `prctl` mode, with the exception that any processes running under `seccomp()` will have the mitigations enabled unconditionally. 
  * `spectre_v2=prctl,ibpb` enables `IBPB` globally in the system, but only enables `STIBP` for processes that have turned it on with `prctl()`. 
  * `spectre_v2=seccomp,ibpb` enables `IBPB` globally, and `STIBP` for all `seccomp()` processes and those that have enabled it explicitly. 

This set contains 28 individual patches; it is not a trivial thing to merge this late in the development cycle (or into a stable kernel update). That appears to be the plan, though; the patches have been pulled into the tip tree and are likely to hit the mainline in the near future. Invasive changes like this are just part of the deal in the post-Spectre world, it seems. Once the dust settles, though, Linux systems will have more complete protection against Spectre variant 2, but the cost of that protection will only need to be paid by those who feel that they need it.
