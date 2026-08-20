---
title: "A formal kernel memory-ordering model (part 1)"
url: https://lwn.net/Articles/718628/
date: "April 14, 2017"
category: Development tools; Memory model
author: "April 14, 2017 By Jade Alglave, Luc Maranget, Paul E. McKenney, Andrea Parri, and Alan Stern"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

April 14, 2017

By Jade Alglave, Luc Maranget, Paul E. McKenney, Andrea Parri, and Alan Stern

It has been said that [`Documentation/memory-barriers.txt`](<https://www.kernel.org/doc/Documentation/memory-barriers.txt>) can be used to [frighten small children](<http://lwn.net/Articles/575835/>), and perhaps this is true. But even if it is true, it is woefully inefficient. After all, there is a huge number of children in this world, so a correspondingly huge amount of time and effort would be required in order to read it to them all. 

This situation clearly calls for automated tooling, which is now available in prototype form; it is now possible to frighten small children at scale. This tool takes short fragments of concurrent C code as input, and exhaustively analyzes the possible results. In other words, instead of perusing `memory-barriers.txt` to find the answer to a memory-ordering question, you can get your answer by writing a small test case and feeding it to the tool, at least for test cases within the tool's current capabilities and limitations. This two-part series gives an introduction to the tool, describing how to use it and how it works. 

To the best of our knowledge, this tool is the first realistic automated representation of Linux-kernel memory ordering, and is also the first to incorporate [read-copy-update (RCU)](<https://www.kernel.org/pub/linux/kernel/people/paulmck/LWNLinuxMM/RCUguarantees.html>) ordering properties. 

This article is organized as follows, with the intended audience for each section in parentheses: 

  1. Why formal memory models? (all). 
  2. Guiding principles (all). 
  3. Causality and ordering (people interested in using memory-ordering tools). 

The second article in this series will look into the issues raised by cycles and how they are important for memory models. It also demonstrates how the [herd](<http://diy.inria.fr>) tool can be used for testing memory models. 

Those wishing to skip the preliminaries found in these two articles and dive directly into the strong model will find it [here](<https://www.kernel.org/pub/linux/kernel/people/paulmck/LWNLinuxMM/StrongModel.html>). (Yes, there is also a [weak model](<https://www.kernel.org/pub/linux/kernel/people/paulmck/LWNLinuxMM/WeakModel.html>), but it is described in terms of how it differs from the strong model. So you should start with the strong model.) 

### Why formal memory models?

Even before Linux, kernel hacking has tended to involve more intuition and less formal methods. Formal methods can nevertheless be useful for providing definite answers to difficult questions. For example, how many different behaviors can a computer program exhibit? Particularly one that uses only values in memory, with no user input or output? 

Computers being the deterministic automata they are, most people would say only one, and for uniprocessor systems they would be basically correct. But multiprocessor systems can give rise to a much wider range of behaviors, owing to subtle variations in the relative timing of the processors and the signals transmitted between them, their caches, and main memory. Memory models try to bring some order to the picture, first and foremost by characterizing exactly which outcomes are possible for a symmetric multiprocessor (SMP) system running a certain (small) program. 

Even better, a _formal_ memory model enables tools to automatically analyze small programs, as described [here](<https://lwn.net/Articles/470681/>) and [here](<https://lwn.net/Articles/608550/>). However, those tools are specialized for specific CPU families. For analyzing the Linux kernel, what we need is a tool targeted at a higher level, one that will be applicable to every CPU architecture supported by the kernel. 

Formal memory models require extreme precision, far beyond what the informal discussion in `memory-barriers.txt` can possibly provide. To bridge this gap in the best way possible, we have formulated the guiding principles listed in the following section. 

### Guiding principles

Our memory model is highly constrained because it must match the kernel's behavior (or intended behavior). However, there are numerous choices to be made, so we formulated the following principles to guide those choices: 

  1. Strength preferred to weakness. 
  2. Simplicity preferred to complexity. 
  3. Support existing non-buggy Linux-kernel code. 
  4. Be compatible with hardware supported by the Linux kernel. 
  5. Support future hardware, within reason. 
  6. Be compatible with the C11 memory model, where prudent and reasonable. 
  7. Expose questions and areas of uncertainty. 

####  Strength preferred to weakness

When all else is equal, a stronger memory model is clearly better, but this raises the question of what is meant by "stronger". For our purposes, one memory model is considered to be stronger than another if it rules out a larger set of behaviors. Thus, the weakest possible memory model is one that would allow a program to behave in any way at all (as exemplified by the "undefined behavior" so common in programming-language standards), whereas the strongest possible memory model is one that says no program can ever do anything. Of course, neither of these extremes is appropriate for the Linux kernel, or for much of anything else. 

The strongest memory model typically considered is [sequential consistency](<https://en.wikipedia.org/wiki/Sequential_consistency>) (SC), and the weakest is release consistency process consistency ([RCpc](<http://dl.acm.org/citation.cfm?id=325102>)). SC prohibits any and all reordering, so that all processes agree on some global order of all processes' accesses, which is theoretically appealing but expensive, so much so that no mainstream microprocessor provides SC by default. In contrast, RCpc is fairly close to the memory models we propose for the Linux kernel, courtesy of the Alpha, ARM, Itanium, MIPS, and PowerPC hardware that the Linux kernel supports. 

On the other hand, we don't want to go overboard. Although strength is preferred over weakness as a general rule, small increases in strength are not worth order-of-magnitude increases in complexity. 

####  Simplicity preferred to complexity

Simpler is clearly better; however, simplicity will always be a subjective notion. A formal-methods expert might prefer a model with a more elegant definition, while a kernel hacker might prefer the model that best matched his or her intuition. Nevertheless, simplicity remains a useful decision criterion. For example, assuming all else is equal, a model with a simpler definition that better matched the typical kernel hacker's intuition would clearly be preferred over a complex counterintuitive model. 

####  Support existing non-buggy Linux-Kernel code

The memory model must support existing non-buggy code in the Linux kernel. However, our model (in its current form) is rather limited in scope. Because it is not intended to be a replacement for either hardware emulators or production compilers, it does _not_ support: 

  * Any number of compiler optimizations. For example, our model currently does not account for compiler optimizations that hoist identical stores from both branches of an `if` statement to precede that statement. (On the other hand, the model also does not cover normal variable access, instead requiring at least `READ_ONCE()` or `WRITE_ONCE()`, each of which greatly limits the compiler's ability to optimize. This restriction is therefore less of a problem than it might at first appear.) 
  * Arithmetic. Not even integer arithmetic! 
  * Multiple access sizes. 
  * Partially overlapping accesses. 
  * Nontrivial data, including arrays and structures. However, trivial linked lists _are_ supported. 
  * Dynamic memory allocation. 
  * Complete modeling of read-modify-write atomic operations. Currently, only atomic exchange is supported. 
  * Locking, though some subset of the Linux kernel's numerous locking primitives is likely be added to a future version. In the meantime, locking may be [emulated using atomic exchange](<https://www.kernel.org/pub/linux/kernel/people/paulmck/LWNLinuxMM/C-locktest.litmus>). 
  * Exceptions and interrupts. 
  * I/O, including DMA. 
  * Self-modifying code, as found in the kernel's alternative mechanism, function tracer, Berkeley Packet Filter JIT compiler, and module loader. 
  * Complete modeling of RCU. For example, we currently exclude asynchronous grace-period primitives such as `call_rcu()` and `rcu_barrier()`. However, we believe that this work includes the first comprehensive formal model of the interaction between RCU reader and synchronous grace periods with memory accesses and memory-ordering primitives. 

**Quick Quiz 1** : But my code contains simple unadorned accesses to shared variables. So what possible use is this memory model to me?   
Answer

As always, adding more detail and functionality to the model slows it down, so the goal is therefore to balance the needs for speed and for functionality. The current model is a starting point, and we hope to incorporate additional functionality over time. We also hope that others will incorporate this memory model into their tools. 

####  Be compatible with hardware supported by the Linux kernel

The memory model must be compatible with the hardware that the Linux kernel runs on. Although the memory model can be (and is) looser than any given instance of hardware, it absolutely must not be more strict. In other words, the memory model must in some sense provide the least common denominator of the guarantees of all memory models of all CPU families that run the Linux kernel. This requirement is ameliorated, to some extent, by the ability of the compiler and the Linux kernel to mask hardware weaknesses. For example: 

  * The Alpha port of the Linux kernel provides memory-barrier instructions as needed to compensate for the fact that Alpha does not respect read-to-read address dependencies. 
  * The Itanium port of GCC emits `ld.acq` for volatile loads and `st.rel` for volatile stores, which compensates for the fact that Itanium does not guarantee read-to-read ordering for normal loads from the same variable. 

Nevertheless, the memory model must be sufficiently weak that it does not rule out behaviors exhibited by any of the CPU architectures the Linux kernel has been ported to. Different CPU families can have quite divergent properties, so that each of Alpha, ARM, Itanium, MIPS, and PowerPC required special attention at some point or another. In addition, hardware memory models are subject to change over time, as are the use cases within the Linux kernel. The Linux-kernel memory model must therefore evolve over time to accommodate these changes, which means that the version presented in this paper should be considered to be an initial draft rather than as being set in stone. It seems likely that this memory model will have the same rate of change as does `Documentation/memory-barriers.txt`. 

Providing compatibility with all the SMP systems supporting Linux is one of the biggest memory-model challenges, especially given that some systems' memory models have not yet been fully defined and documented. In each case, we have had to take our best guess based on existing documentation, consultation with those hardware architects willing to consult, formal memory models (for those systems having them), and experiments on real hardware, for those systems we have access to. In at least one case, this might someday involve a computer museum. 

Thankfully, this situation has been improving. For example, although formal memory models have been available for quite some time (such as [here [400-page PDF]](<http://www.hpl.hp.com/techreports/Compaq-DEC/WRL-95-9.pdf>)), tools that apply memory models to litmus tests have only appeared much more recently. We most certainly hope that this trend toward more accessible and better-defined memory models continues, but in the meantime we will continue to work with whatever is available. 

####  Support future hardware, within reason

The memory model should support future hardware, within reason. Linux-kernel ports to new hardware must supply their own code for the various memory barriers, and might one day also need to supply their own code for similar common-code primitives. But since common code is valuable, an architecture wishing to supply its own code for (say) `READ_ONCE()` will need a very good reason for doing so. 

This proposal assumes that future hardware will not deviate too far from current practice. For example, if you are porting Linux to a quantum supercomputer, the memory model is likely to be the least of your worries. 

####  Be compatible with the C11 memory model, where prudent and reasonable

Where possible, the model should be compatible with the existing C and C++ memory models. However, there are a number of areas where it is necessary to depart from these memory models: 

  * The `smp_mb()` full memory barrier is stronger than that of C and C++. But let's face it, `smp_mb()` was there first, and there is a lot of code in the kernel that might be adapted to `smp_mb()`'s current semantics. 
  * The Linux kernel's value-returning read-modify-write atomics feature ordering properties that are not found in their C/C++ counterparts. 
  * The `smp_mb__before_atomic()`, `smp_mb__after_atomic()`, and `smp_mb__after_unlock_lock()` barrier-amplification APIs have no counterparts in the C/C++ API. 
  * The `smp_read_barrier_depends()` macro does not have a direct equivalent in the C/C++ memory model. 
  * The Linux kernel's notion of control dependencies does not exist in C/C++. However, control dependencies are an important example of instruction ordering, so the memory model must account for them. 
  * The Linux-kernel notion of RCU grace periods does not exist in C/C++. (However, the RCU-related proposals [P0190R3 [PDF]](<http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0190r3.pdf>), [P0461R1 [PDF]](<http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0461r1.pdf>), [P0462R1 [PDF]](<http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0462r1.pdf>), [P0561R0 [PDF]](<http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0561r0.html>), and [P0566R0 [PDF]](<http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0566r0.pdf>) are being considered by the committee.) 

On the positive side, the Linux kernel has recently been adding functionality that is closer to that of C and C++ atomics, with the ongoing move from `ACCESS_ONCE()` to `READ_ONCE()` and `WRITE_ONCE()` being one example and the addition of `smp_load_acquire()` and `smp_store_release()` being another. 

####  Expose questions and areas of uncertainty

Defining a memory model inevitably uncovers interesting questions and areas of uncertainty. For example: 

  * The Linux-kernel memory model is [more strict than that of C11](<http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0124r1.html>). It is useful to flag the differences in order to alert people who might otherwise be tempted to rely solely on C11. It is also quite possible that some of the Linux kernel's strictness is strictly historical, in which case it might (or might not) be worth considering matching C11 semantics for those specific situations. 
  * Release-acquire chains are required to provide ordering to those tasks participating in the chain. Failure to provide such ordering would have many problematic consequences, not least being that locking would not work correctly. For tasks external to the chain, ordering cannot be provided for a write preceding the first release and a read following the last acquire due to hardware limitations. For example, if one process writes to variable `x` while holding a lock and a later critical section for that same lock reads from variable `y`, the read of `y` might execute before the write of `x` has propagated to an unrelated process not holding the lock. 
  * It turns out that release-acquire chains can be implemented using `READ_ONCE()` instead of `smp_load_acquire()`. (However, substituting `WRITE_ONCE()` for `smp_store_release()` does _not_ work on all architectures.) Should the model require the use of `smp_load_acquire()`? 
  * Some architectures can "erase" writes, so that ordering specific to a given write might not apply to a later write to that same variable by that same task, even though coherence ordering would normally order the two writes. This can give rise to bizarre results, such as the possible outcomes of a code sequence depending on the code that follows it. (However, such results appear to be restricted to litmus tests that can best be described as "rather strange".) 
  * One interesting corner case of hardware memory models is that weak barriers (i.e., `smp_wmb()`) suffice to provide transitive orderings when all accesses are writes. However, we were unable to come up with reasonable use cases, and furthermore, the things that looked most reasonable proved to be attractive nuisances. Should the memory model nevertheless provide ordering in this case? (If you know of some reason why this ordering should be respected by the memory models, please don't keep it a secret). 

In a perfect world, we would resolve each and every area of uncertainty, then produce a single model reflecting full knowledge of all the hardware that the Linux kernel supports. However, astute readers might have noticed that the world is imperfect. Furthermore, rock-solid certainties can suddenly be cast into doubt, either with the addition of an important new architecture or with the uncovering of a misunderstanding or an error in documentation of some existing architecture. It will therefore be sometimes necessary for the Linux kernel memory model to say "maybe". 

Unfortunately, existing software tools are unable to say "maybe" in response to a litmus test. We therefore constructed not one but two formal models, one strong and the other less strong. These two models will disagree in "maybe" cases. Kernel hackers should feel comfortable relying on ordering only in cases where both models agree that ordering should be provided, and hardware architects should feel the need to provide strong ordering unless both models agree that strong ordering need not be provided. (Currently these models are still very much under development, so it is still unwise to trust either model all that much.) 

###  Causality and ordering

Causality is an important property of memory models, in part because causality looms large in most peoples' intuitive understanding of concurrent code. However, causality is a generic term, lacking the precision required for a formal memory model. In this series we will therefore use the terms "causality" and "causal relationship" quite sparingly, instead defining precise terms that will be used directly within the memory model. But a brief discussion now will help illuminate the topic and will introduce some important relationships between causality, ordering, and memory models. 

_Causality_ is simply the principle that a cause happens before its effect, not after. It is therefore a statement about ordering of events in time. Let's start with the simplest and most direct example. If CPU A writes a value to a shared variable in memory, and CPU B reads that value back from the shared variable, then A's write must execute before B's read. This truly is an example of a cause-and-effect relation; the only way B can possibly know the value stored by A is to receive some sort of message sent directly or indirectly by A (for example, a cache-coherence protocol message). 

Messages take time to propagate from one CPU or cache to another, and they cannot be received before they have been sent. (In theory, B could guess the value of A's write, act on that guess, check the guess once the write message arrived, and if the guess was wrong, cancel any actions that were inconsistent with the actual value written. Nevertheless, B could not be entirely certain that its guess is correct until the message arrives--and our memory models assume that CPUs do not engage in this sort of guessing, at least not unless they completely hide its effects from the software they are running.) 

On the other hand, if B does not read the value stored by A but rather an earlier value, then there need not be any particular temporal relation between A's write and B's read. B's read could have executed either before or after A's write, as long as it executed before the write message reached B. In fact, on some architectures, the read could return the old value even if it executed a short time _after_ the message's arrival. _A fortiori_ , there would be no cause-and-effect relation. 

Another example of ordering also involves the propagation of writes from one CPU to another. If CPU A writes to two shared variables, these writes need not propagate to CPU B in the same order as the writes were executed. In some architectures it is entirely possible for B to receive the messages conveying the new values in the opposite order. In fact, it is even possible for the writes to propagate to CPU B in one order and to CPU C in the other order. The only portable way for the programmer to enforce write propagation in the order given by the program is to use appropriate memory barriers or barrier-like constructs, such as `smp_mb()`, `smp_store_release()`, or C11 non-relaxed atomic operations. 

A third example of ordering involves events occurring entirely within a single CPU. Modern CPUs can and do reorder instructions, executing them in an order different from the order they occur in the instruction stream. There are architectural limits to this sort of thing, of course.  Perhaps the most pertinent for memory models is the general principle that a CPU cannot execute an instruction before it knows what that instruction is supposed to do. 

For example, consider the statement "`x = y;`". To carry out this statement, a CPU must first load the value of `y` from memory and then store that value to `x`. It cannot execute the store before the load; if it tried then it would not know what value to store. This is an example of a _data dependency_. There are also _address dependencies_ (for example, "`a[n] = 3;`" where the value of `n` must be loaded before the CPU can know where to store the value 3). Finally, there are _control dependencies_ (for example, "`if (i == 0) y = 5;`" where the value of `i` must be loaded before the CPU can know whether to store anything into `y`). In the general case where no dependency is present, however, the only portable way for the programmer to force instructions to be executed in the order given by the program is to use appropriate memory barriers or barrier-like constructs. 

Finally, at a higher level of abstraction, source code statements can be reordered or even eliminated entirely by an optimizing compiler. We won't discuss this very much here; `memory-barriers.txt` contains a number of examples demonstrating the sort of shenanigans a compiler can get up to when translating a program from source code to object code. 

### To be continued

[The second half of this series](<https://lwn.net/Articles/720550/>) focuses on the specific problem of cycles. 

### Acknowledgments

We owe thanks to H. Peter Anvin, Will Deacon, Andy Glew, Derek Williams, Leonid Yegoshin, and Peter Zijlstra for their patient explanations of their respective systems' memory models. We are indebted to Peter Sewell, Susmit Sarkar, and their groups for their seminal work formalizing many of these same memory models. We all owe thanks to Dmitry Vyukov, Boqun Feng, and Peter Zijlstra for their help making this human-readable. We are also grateful to Michelle Rankin and Jim Wasko for their support of this effort. 

####  Answer to the Quick Quiz

**Quick Quiz 1** : But my code contains simple unadorned accesses to shared variables. So what possible use is this memory model to me? 

**Answer** : You are of course free to use simple unadorned accesses to shared variables in your code, but you are then required to make sure that the compiler isn't going to trip you up--as has always been the case. Once you have made sure that the compiler won't trip you up, simply translate those accesses to use `READ_ONCE()` and `WRITE_ONCE()` when using the model. Of course, if your code gives the compiler the freedom to rearrange your memory accesses, you may need multiple translations, one for each possible rearrangement. 

**Back to Quick Quiz 1**.
