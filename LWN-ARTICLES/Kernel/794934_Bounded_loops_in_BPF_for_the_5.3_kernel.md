---
title: Bounded loops in BPF for the 5.3 kernel
url: https://lwn.net/Articles/794934/
date: "July 31, 2019"
category: "BPF-Loops"
author: "July 31, 2019 This article was contributed by Marta Rybczyńska"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

July 31, 2019

This article was contributed by Marta Rybczyńska

BPF programs have gained significantly in capabilities over the last few years and can now perform many useful operations. That said, BPF developers have had to work around an annoying limitation until recently: they could not use loops. This restriction was recently lifted by a [patch set](<https://lwn.net/ml/netdev/20190615191225.2409862-1-ast@kernel.org/>) from Alexei Starovoitov that was merged for Linux 5.3. In addition to adding support for loops, it also greatly decreases the load time of most BPF programs.

#### The problem

Before a BPF program runs, it needs to be checked to ensure that it cannot cause harm to the system. For example, if it does not complete in a bounded time, it could be used to carry out a denial-of-service attack on the system. The task of checking programs is performed by the BPF verifier. Until recently, the verifier could not handle loops, meaning that all programs with loops were rejected.

Since loops are one of the basic ingredients of a computer program, this limitation was often hit by developers, who worked around it by unrolling loops (either by hand or using a compiler pragma). It comes as no surprise that there have been a number of attempts to change the situation by allowing loops that provably have a limited (and reasonable) number of iterations. Those include a [proof-of-concept proposal](<https://lwn.net/Articles/748032/>) from Edward Cree. [Another attempt](<https://lwn.net/ml/netdev/20180601092646.15353.28269.stgit@john-Precision-Tower-5810/>) by John Fastabend tried to solve the problem with loop analysis: identifying each loop's induction variable and verifying its use. Fastabend covered the theoretical background and the possible solutions in [a presentation at the 2018 Linux Plumbers Conference](<https://lwn.net/Articles/773605/>). The kernel developers decided not to take either solution at that time.

In parallel, important work has been put into optimizing the BPF verifier, as shown in a set of [Linux Storage, Filesystem and Memory Management Summit slides](<https://docs.google.com/presentation/d/1oaSa_dlInrTKMzP9_fYgLHN_I5j8pDrD0j3Y93inWiQ/edit#slide=id.g58e9ace034_0_0>) by Jakub Kicinski. One important outcome of this work was increasing the size limitation for BPF programs in the 5.2 kernel; instead of 4096 instructions, a program can execute up to one million. These optimizations allow a more direct — even brute-force — approach to the bounded-loop-checking problem: instead of adding special handling for loops, the verifier can simply simulate the iterations of a loop as a collection of states no different from any others. 

#### The BPF verifier

When the BPF verifier analyzes a program, it creates a model in the form of a state machine. It checks each state for incorrect behavior; while it does so, it records the states it has already verified. Later on, when it reaches a state that is equivalent to one that was already verified, it can conclude there is nothing more to do on that path, which can thus be pruned. This pruning significantly reduces the amount of work that the verifier must do. 

State-based pruning is essential for verifier performance, but it imposes its own cost in the form of comparing states and copying the safe ones. Recent analysis of a set of networking-related BPF programs showed that 80% of the saved states will never be matched and, thus, will never prune a future search. As a result, there may be performance gains to be had by reducing the number of states and pruning points maintained by the verifier. 

The old optimizer worked by placing pruning points both before and after each jump instruction, resulting in pruning points being added approximately every four instructions. It turns out that simply placing a pruning point every ten instructions, regardless of the instruction type, improves performance considerably. The verifier was also modified to aggressively drop saved states that do not actually prune paths. These changes improve verifier performance by up to 20% and increase the length of a program that can be verified in a reasonable time by one-third. 

#### Adding bounded loops

Starovoitov's patch set introducing bounded loops builds on that earlier work. It also includes a number of other improvements. This is partly the result of a big performance regression introduced by the first patch in the series that is a necessary building block for the final solution.

That regression takes the form of a performance degradation when adding variable tracking on the stack. The compiler often puts variables on the stack (or "spills" them) when it needs to free up some registers. The verifier should be able to track such variables, since it may need to make decisions based on their contents. Until now it did not do so, resulting in certain programs being incorrectly rejected. It turns out that loop induction variables are often spilled, so fixing the tracking of their contents was necessary to be able to verify loop termination. 

On the other hand, tracking specific variable values (as opposed to ranges of possible values) creates more states, decreasing the effectiveness of state pruning; tracking them on the stack makes the problem even worse. The number of states increases, so the verification time also increases. When debugging this issue, Starovoitov found another effect; the performance penalty is aggravated by changes in the Clang compiler. It turns out that newer Clang versions spill fewer variables onto the stack, reducing state pruning even in the absence of complete value tracking. The two problems together caused an important degradation of verifier performance, with the exact results available in the [commit message](<https://lwn.net/ml/netdev/20190615191225.2409862-2-ast@kernel.org/>).

Another feature needed by the bounded-loop support is extending the way the verifier handles conditional branches. If a comparison takes place between two constants, the verifier can easily determine which branch will be taken. Comparisons involving variable values are clearly harder, which is why, until now, the verifier supported only comparisons of a register with a constant. In this patch set, Starovoitov [added support](<https://lwn.net/ml/netdev/20190615191225.2409862-4-ast@kernel.org/>) for tests comparing two registers as well. This enhancement was necessary to be able to simulate the execution of loop tests. 

The third and final piece consists of adding a parent-child relationship between the states. When the verifier needs to explore two branches in the program, it considers them both as child states of the parent state. It also counts the number of branches to explore so that it knows how many are left. With those features, and more aggressive heuristics in the exploration state pruning, the verifier can support bounded loops. It does it simply by simulating all possible iterations of their execution.

With bounded-loop support added, only one item remained: the regression introduced at the beginning of the series. The solution comes in the [last patch of the series](<https://lwn.net/ml/netdev/20190615191225.2409862-10-ast@kernel.org/>). Based on the improvements added before (especially the parentage tracking), Starovoitov added tracking of precise scalar values. Those values are stored in registers and will be modified during program execution. The verifier needs to have precise values to analyze branches correctly, but it need not track the value of _every_ register precisely; only those that control branching require that precision. So the verifier does not incur the cost of tracking all registers precisely; instead, when the need arises, it backtracks through the use of a register in the code to generate a precise value if possible. 

#### Summary

The BPF verifier has undergone a number of changes in this patch set. The resulting code not only adds support for bounded loops, but also a number of important optimizations. Writing BPF programs for the kernel should be rather easier in the 5.3 release, though undoubtedly BPF developers will still have ample opportunity to complain about the remaining hoops they have to jump through to convince the verifier that their programs are safe.
