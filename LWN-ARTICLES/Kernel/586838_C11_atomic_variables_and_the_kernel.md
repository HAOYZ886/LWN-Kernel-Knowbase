---
title: C11 atomic variables and the kernel
url: https://lwn.net/Articles/586838/
date: "February 18, 2014"
category: C11 atomic operations
author: "By Jonathan Corbet February 18, 2014"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
February 18, 2014

The C11 standard added a number of new features for the C and C++ languages. One of those features — built-in atomic types — seems like it would naturally be of interest to the kernel development community; for the first time, the language standard tries to address concurrent access to data on contemporary hardware. But, as recent discussions show, it may be a while before C11 atomics are ready for use with the kernel — if they ever are — and the kernel community may not feel any great need to switch. 

The kernel provides a small set of atomic types now, along with a set of operations to manipulate those types. Kernel atomics are a useful way of dealing with simple quantities in an atomic manner without the need for explicit locking in the code. C11 atomics should be useful for the implementation of the kernel's atomic types, but their scope goes beyond that application. 

In particular, each access to a C11 atomic variable has an explicit "memory model" associated with it. Memory models describe how accesses to memory can be optimized by the processor or the compiler; the more relaxed models can allow operations to be reordered or combined for improved performance. The default model ("sequentially consistent") is the strictest; it does not allow any combining or reordering of operations in any way that would be visible anywhere else in the program. The problem with this model is that it is quite expensive, and, most of the time, that expense does not need to be incurred for correct operation. The more relaxed models exist to allow for optimizations to be performed in a controlled manner while ensuring correct ordering when needed. 

Thus, C11 atomic variables include features that, in the kernel, are usually implemented with memory barriers. So, for example, in current kernel code, one could see something like: 

```
smp_store_release(&x, new_value);
```

The `smp_store_release()` barrier (described in more detail in [this article](<https://lwn.net/Articles/576486/>)) tells the processor to ensure that any reads or writes executed before this assignment are visible on all processors before the assignment to `x` becomes visible. Reordering of operations that all occur before this barrier is still possible, as is the reordering of operations that all occur afterward. In most code, quite a bit of reordering can take place without affecting the correctness of the result. The use of explicit barriers in places where ordering _does_ matter enables most accesses to be performed without barriers, enabling optimization and improving performance significantly. 

If, instead, `x` were a C11 atomic type, one might write: 

```
atomic_store(&x, new_value, memory_order_release);
```

Where `memory_order_release` specifies the same ordering requirements as `smp_store_release()`. (See [this page](<http://en.cppreference.com/w/cpp/atomic/memory_order>) for a description of the C11 memory models). 

If the `memory_order_relaxed` model (which imposes no ordering requirements on the access) is used for surrounding accesses to other atomic variables where ordering is not important, the end result should be similar to that achieved with `smp_store_release()`. But the former version is implemented with tricky, architecture-specific code within the kernel; the latter version, instead, causes the desired code to be emitted directly by the compiler. 

When the kernel first gained support for multiprocessor systems, the C language had no concept of atomic types or memory barriers, so the kernel developers naturally had to create their own. Now that the language standard has caught up, one might think that changing the kernel to make use of the standard atomic types would make sense. And, someday, it might, but that transition is likely to be slow and fitful at best. 

#### Optimization worries

The problem is that compilers tend to be judged on the speed of the code they generate, so compiler developers have a strong incentive to optimize code to the greatest extent possible. Sometimes those optimizations can break code that is not written with an attentive eye toward the standard; the kernel developers' perspective is that compiler developers will often rely on a legalistic reading of standards to justify "optimizations" that (from the kernel developer's viewpoint) make no sense and break code needlessly. Highly concurrent code, as is found in the kernel, tends to be more susceptible to optimization-caused problems than just about anything else. So kernel developers have learned to be careful. 

One of the scariest potential problems is "speculative stores," where an incorrect value becomes visible on a temporary basis. A classic example would be code like this: 

```
if (x)
    	y = 1;
        else
    	y = 2;
```

It would not be uncommon for a compiler to optimize this code by turning it into something like this: 

```
y = 2;
        if (x)
    	y = 1;
```

For sequential code operating in its own address space, the end result is the same, and the latter version avoids one jump. But if `y` is visible elsewhere, the value stored speculatively before the test may be seen by code that will proceed to do the wrong thing, causing things to go off the rails. Clearly, optimizations that cause incorrect values to become visible to any running thread must be avoided if the system is to run correctly. 

When David Howells recently [suggested](<https://lwn.net/Articles/586850/>) that C11 atomic variables could be used in the kernel, speculative stores were one of the first concerns to be raised. The behavior of atomic variables as described by the standard is complex, to put it lightly, and there were real worries that the standard could allow compilers to generate speculative writes. An extensive and sometimes colorful discussion put most of those concerns to rest, but Paul McKenney, who has been representing the kernel's interests within the standard committee, is [still not completely sure](<https://lwn.net/Articles/586851/>): 

From what I can see at the moment, the standard -generally- avoids speculative stores, but there are a few corner cases where it might allow them. I will be working with the committee to see exactly what the situation is. 

Another area of concern is control dependencies: situations where atomic variables and control flow interact. Consider a simple bit of code: 

```
x = atomic_load(&a, memory_order_relaxed);
        if (x)
      	atomic_store(&y, 42, memory_order_relaxed);
```

The setting of `y` has a control dependency on the value of `x`. But the C11 standard does not currently address control dependencies at all, meaning that the compiler or processor could play with the order of the two atomic operations, or even try to optimize the branch out altogether; see [this explanation from GCC developer Torvald Riegel](<https://lwn.net/Articles/586854/>) for details. Again, the results of this kind of optimization in the kernel context could be disastrous. 

For cases like this, Paul [suggested](<https://lwn.net/Articles/586856/>) that some additional source-code markup and a new `memory_order_control` memory model could be used in the kernel to make the control dependency explicit: 

```
x = atomic_load(&a, memory_order_control);
        if (control_dependency(x))
      	atomic_store(&b, 42, memory_order_relaxed);
```

But this approach is unlikely to be taken, given [just how unhappy](<https://lwn.net/Articles/586857/>) Linus was with the idea. From his point of view, the control dependency should be obvious — the code is testing the value of `x`, after all. Any compiler that would move the `atomic_store()` operation in an externally visible way, he said, is simply broken. 

There has also been some concern about "value speculation," wherein the compiler guesses that a variable will have a specific value and inserts a branch to fix things up if the guess is wrong. The processor's branch prediction hardware will then, hopefully, speed things up in cases where the guess is correct. See [this note from Paul](<https://lwn.net/Articles/586975/>) for an example of how value speculation might work — and how it might get things wrong. The good news on this front is that it seems that this kind of speculation will not be allowed. But it is not 100% clear that the current standard forbids it in all cases. 

#### Non-local optimizations considered harmful

Yet another concern is global optimization. Compiler developers are increasingly trying to optimize programs at the level of entire source files, or even larger groups of files. This kind of optimization can work well as long as the compiler truly understands how variables are used. But the compiler is not required to understand the real hardware that the program is running on; it is, instead, required to prove its decisions against a virtual machine defined by the standard. If the real computer behaves in ways that differ from the virtual machine, things can go wrong. 

Consider [this example](<https://lwn.net/Articles/586957/>) raised by Linus: the compiler might look at how the kernel accesses page table entries and notice that no code ever sets the "page dirty" bit. It might then conclude that any tests against that bit could simply be optimized out. But that bit _can_ change; it's just that the hardware makes the change, not the kernel code. So any optimizations made based on the notion that the compiler can "prove" that bit will never be set will lead to bad things. Linus concluded: ""Any optimization that tries to prove anything from more than local state is by definition broken, because it assumes that everything is described by the program."" 

Paul sent out [a list of other situations](<https://lwn.net/Articles/586962/>) where the compiler's virtual machine model might not match what is really happening. His examples included assembly code, kernel modules (which can access exported symbols, but which might not even exist when the compiler is making its decisions), kernel-space memory mapped into user space, [JIT-compiled BPF code](<https://lwn.net/Articles/437981/>), and ""probably other stuff as well"". In short, there is a lot going on inside a kernel that the compiler cannot be expected to know about. 

One solution to many of these non-local problems is to use `volatile` with the affected variables. Simply identifying such variables would be an error-prone exercise, of course, but there is a worse problem: using `volatile` turns off all optimization for the affected variable, defeating the purpose of using atomic variables in the first place. If `volatile` must be used, the kernel is better off staying with its current memory barrier scheme, which is designed to allow as much compiler- and processor-level optimization as possible, but no more than that. 

Will it come to that? Despite his worries, Linus has actually expressed some confidence that real-world compilers will not break things badly: 

In *practice*, I seriously doubt any reasonable compiler can actually make a mess of it. The kinds of optimizations that would actually defeat the dependency chain are simply not realistic. And I suspect that will end up being what we rely on - there being no actual sane sequence that a compiler would ever do, even if we wouldn't have guarantees for some of it. 

But he has also been clear that his trust of compiler developers only goes so far and that, if necessary, the kernel community is more than prepared to stick with its current approach, which, he [said](<https://lwn.net/Articles/586974/>), is ""generally *fine*"". 

#### Does the kernel need C11 atomics?

Linus [went on](<https://lwn.net/Articles/586858/>) to make it clear that he is serious about this; if atomic variables as found in the C11 standard and its implementations do not provide what the kernel wants, the kernel will simply not use that feature. The kernel project, he said, is in a fairly strong bargaining position when it comes to atomic variables: 

And the thing is, I suspect that the Linux kernel is the most complete - and most serious - user of true atomics that the C11 people can sell their solution to. 

If we don't buy it, they have no serious user. Sure, they'll have lots of random other one-off users for their atomics, where each user wants one particular thing, but I suspect that we'll have the only really unified portable code base that handles pretty much *all* the serious odd cases that the C11 atomics can actually talk about to each other. 

On the other hand, he said, the solutions found in the kernel now work just fine; there is no real need to move away from them if the kernel community does not want to. 

In truth, there may well be other serious users; the GNU C library [is using C11 atomics](<https://lwn.net/Articles/586859/>) now for a few architectures, for example. And, while Torvald [agreed](<https://lwn.net/Articles/586860/>) that the kernel could continue to use its own solution, he also pointed out that there would be some advantages to using the standard mechanism. The widespread testing that this mechanism will receive was at the top of his list. One could also note that the kernel's tricky, architecture-specific barrier code could conceivably go away, replaced by more widely used code maintained by the compiler developers. That code would also, hopefully, be less likely to break when new releases of the compiler come out. 

Beyond that, Torvald [pointed out](<https://lwn.net/Articles/586862/>), C11 atomics can benefit from a fair amount of academic work that has been done. Some researchers at the University of Cambridge have [come up with a formal description [PDF]](<http://www.cl.cam.ac.uk/~mjb220/popl085ap-sewell.pdf>) of how C11 concurrency should work. Associated with that description is [an interactive memory model simulator](<http://svr-pes20-cppmem.cl.cam.ac.uk/cppmem/>) that can test code snippets for race conditions. And, in the end, if a large number of programs make use of C11 atomics, that should result in the quality of compiler implementations improving quickly. 

Finally, if C11 atomic variables can be made to work in real-world programs, they could go a long way toward the establishment of reliable patterns for how C (and C++) can be used in concurrent environments. At the moment, there is no way for developers to know what is safe to do — now, and in the future. As Peter Sewell (one of the above-mentioned Cambridge researchers) [put it](<https://lwn.net/Articles/586977/>): 

There are too many compiler optimisations for people to reason directly in terms of the set of all transformations that they do, so we need some more concise and comprehensible envelope identifying what is allowed, as an interface between compiler writers and users. 

The C11 standard is meant to be that "envelope," though, as Peter admitted, it is ""not yet fully up to that task"". But if the remaining uncertainties and problems can be addressed, C11 atomics could become a common language with which developers can reason about concurrency and allowable optimizations. Developers might come to understand the issues better, and kernel code might become a bit more widely accessible to developers who understand the standard. 

So it might well benefit the kernel to make use of this relatively new language feature. Nobody has closed the door on that possibility, but any transition in that direction will require a lot of time, testing, and confidence building. Bugs resulting from low-level concurrency management problems can be among the hardest to find, reproduce, or diagnose; nobody will be in a hurry to replace the kernel's atomics and memory barriers without a high level of assurance that the change will not result in the introduction of that kind of issue.
