---
title: Injecting speculation barriers into BPF programs
url: https://lwn.net/Articles/1018494/
date: "May 5, 2025"
category: "BPF-Security; Security-Meltdown and Spectre"
author: "By Jonathan Corbet May 5, 2025"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
May 5, 2025

The disclosure of the [Spectre class of hardware vulnerabilities](<https://en.wikipedia.org/wiki/Spectre_\(security_vulnerability\)>) created a lot of pain for kernel developers (and many others). That pain was especially acutely felt in the BPF community. While an attacker might have to painfully search the kernel code base for exploitable code, an attacker using BPF can simply write and load their own speculation gadgets, which is a much more efficient way of operating. The BPF community reacted by, among other things, disallowing the loading of programs that may include speculation gadgets. Luis Gerhorst would like to change that situation with [this patch series](<https://lwn.net/ml/all/20250421091802.3234859-1-luis.gerhorst@fau.de>) that takes a more direct approach to the problem. 

While the potential to enable speculative-execution attacks may be a concern for any BPF program, the problem is especially severe for unprivileged programs — those that can be loaded by ordinary users. Most program types require privilege but there are a couple of packet-filter program types that do not (though the `unprivileged_bpf_disabled` sysctl knob can disable those types too). Among the many defenses added to the BPF subsystem is [this patch by Daniel Borkmann](<https://git.kernel.org/linus/9183671af6db>), which was merged for the 5.13 release in 2021. It causes the verifier to treat possible speculative paths (for [Spectre variant 1](<https://en.wikipedia.org/wiki/Spectre_\(security_vulnerability\)#Spectre_Variant_1>) in particular) as real alternatives when simulating the execution of a program, even though the verifier can demonstrate that such paths will not be taken in non-speculative execution. If the program does something untoward on one of those speculative paths, it will be rejected by the verifier. 

In other words, an unprivileged BPF program must behave correctly even in the presence of branch decisions that have been guessed incorrectly by the CPU. Gerhorst performed an analysis of 364 BPF programs from various projects, and found that ""31% to 54% of programs"" would be rejected as a result of this extra verification requirement. So a sizable subset of valid BPF programs cannot be loaded by unprivileged users out of fear of speculative attacks. That makes BPF rather less useful than it would otherwise be. 

The approach chosen by Gerhorst to address this problem is relatively simple: if the verifier is unable to prove that a given speculative path will execute correctly, it injects a speculation barrier into the code at the branch that might be mispredicted. That barrier (an `lfence` instruction on x86_64 systems) halts all speculation until the true execution catches up with the barrier, making it impossible to mispredict the branch. The verification of the speculative path can simply be halted, since that path will not be taken even speculatively; any bad behavior in that path will thus no longer cause the program to fail verification. 

The solution is conceptually simple, though it requires some complexity in the verifier to implement. It does widen the set of programs that are now acceptable for an unprivileged user to load (though Gerhorst does not say by how much). There are some downsides that are part of this solution as well, though. 

One of those is performance; speculation barriers are, since they disable speculative execution, relatively expensive. The actual cost of injecting them, according to Gerhorst, is ""0% to 62%"", depending on the program. This cost is only paid, though, by programs that would be rejected by the verifier without the barrier injection; users are likely to agree that slowed-down execution is, in the end, better than being unable to run the program at all. 

The other potential concern is for architectures that are vulnerable to Spectre variant 1, but which do not provide a suitable barrier instruction. This patch series, as currently written, will disable the current, verifier-based checking on those architectures without actually replacing it with barrier-based protection. According to Gerhorst (as described in [this patch](<https://lwn.net/ml/all/20250421091802.3234859-9-luis.gerhorst@fau.de>)), the only architecture that would be affected by this problem is MIPS, which does not allow unprivileged BPF at all by default. Thus, he said, this potential security regression is ""deemed acceptable"". The status of some of the more obscure architectures supported by the kernel, including whether they are vulnerable to Spectre variant 1 at all, is unknown, he said. 

This is the second posting for this series; the [first posting](<https://lwn.net/ml/all/20250224203619.594724-1-luis.gerhorst@fau.de/>) was in late February. So far, neither posting has received any review comments at all. That may be because the BPF community has long since [decided](<https://lwn.net/Articles/796328/>) that unprivileged BPF is a dead end. It is simply too difficult to make it possible for unprivileged users to load code to run in the kernel and have the result be secure. The Spectre vulnerabilities may have been the last straw in that regard; they opened up a whole new set of exploitation paths that the BPF developers had never needed to take into account before. 

Still, unprivileged BPF does exist in current kernels, though it is not clear how many distributors enable it. If the kernel is going to support that feature, it should be supported as well as possible. Gerhorst's patches make it possible to run more programs in the unprivileged mode without opening up Spectre vulnerabilities. The goal seems worthy; the real question will be whether it justifies the complexity that the implementation necessarily adds to the verifier.
