---
title: Rethinking optimization for size
url: https://lwn.net/Articles/534735/
date: "January 30, 2013"
category: GCC
author: "By Jonathan Corbet January 30, 2013"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
January 30, 2013

Contemporary compilers are capable of performing a wide variety of optimizations on the code they produce. Quite a bit of effort goes into these optimization passes, with different compiler projects competing to produce the best results for common code patterns. But the nature of current hardware is such that some optimizations can have surprising results; that is doubly true when kernel code is involved, since kernel code is often highly performance-sensitive and provides an upper bound on the performance of the system as a whole. A recent discussion on the best optimization approach for the kernel shows how complicated the situation can be. 

Compiler optimizations are often aimed at making frequently-executed code (such as that found in inner loops) run more quickly. As an artificially simple example, consider a loop like the following: 

```
for (i = 0; i < 4; i++)
    	do_something_with(i);
```

Much of the computational cost of a loop like this may well be found in the loop structure itself — incrementing the counter, comparing against the maximum, and jumping back to the beginning. A compiler that performs loop unrolling might try to reduce that cost by transforming the code into something like: 

```
do_something_with(0);
        do_something_with(1);
        do_something_with(2);
        do_something_with(3);
```

The loop overhead is now absent, so one would expect this code to execute more quickly. But there is a cost: the generated code may well be larger than it was before the optimization was applied. In many situations, the performance improvement may outweigh the cost, but that may not always be the case. 

GCC provides an optimization option (`-Os`) with a different objective: it instructs the compiler to produce more compact code, even if there is some resulting performance cost. Such an option has obvious value if one is compiling for a space-constrained environment like a small device. But it turns out that, in some situations, optimizing for space can also produce _faster_ code. In a sense, we are all running space-constrained systems, in that the performance of our CPUs depends heavily on how well those CPUs are using their cache space. Space-optimized code can make better use of scarce instruction cache space, and, as a result, perform better overall. With this in mind, compilation with `-Os` was [made generally available](<http://git.kernel.org/linus/0910b444bcd7ea7f437a3ff48e0e3d2495ead22e>) for the 2.6.15 kernel in 2005 and [made non-experimental](<http://git.kernel.org/linus/96fffeb4b413a4f8f65bb627d59b7dfc97ea0b39>) for 2.6.26 in 2008. 

Unfortunately, `-Os` has not always lived up to its promise in the real-world. The problem is not necessarily with the idea of creating compact code; it has more to do with how GCC interprets the `-Os` option. In the space-optimization mode, the compiler tends to choose some painfully slow instructions, especially on older processors. It also discards the branch prediction information provided by kernel developers in the form of the `likely()` and `unlikely()` macros. That, in turn, can cause rarely executed code to share cache space with hot code, effectively wasting a portion of the cache and wiping out the benefits that optimizing for space was meant to provide. 

Because `-Os` did not produce the desired results, Linus [disabled it by default](<http://git.kernel.org/linus/281dc5c5ec0fb299514567cbc358562649c1af95>) in 2011, effectively ending experimentation with this option. Recently, though, Ling Ma posted [some results](<https://lwn.net/Articles/534745/>) suggesting that the situation might have changed. Recent Intel processors, it seems, have a new cache for decoded instructions, increasing the benefit obtained by having code fit into the cache. The performance of the repeated "move" instructions used by GCC for memory copies in `-Os` mode has also been improved in newer processors. The posted results claim a 4.8% performance improvement for the netperf benchmark and 2.7% for the volano benchmark when `-Os` is used on a newer CPU. Thus, it was suggested, maybe it is time to reconsider `-Os`, at least for some target processors. 

Naturally, the situation not quite that simple. Valdis Kletnieks [complained](<https://lwn.net/Articles/534746/>) that the benchmark results may not be showing an actual increase in real-world performance. Distributors hate shipping multiple kernels, so an optimization mode that only works for some portion of a processor family is unlikely to be enabled in distributor kernels. And there is still the problem of the loss of branch prediction information which, as Linus [verified](<https://lwn.net/Articles/534749/>), still happens when `-Os` is used. 

What is really needed, it seems, is a kernel-specific optimization mode that is more focused on instruction-cache performance than code size in its own right. This mode would take some behaviors from `-Os` while retaining others from the default `-O2` mode. Peter Anvin [noted](<https://lwn.net/Articles/534750/>) that the GCC developers are receptive to the idea of implementing such a mode, but there is nobody who has the time and inclination to work on that project at the moment. It would be nice to have a developer who is familiar with both the kernel and the compiler and who could work to make GCC produce better code for the kernel environment. Until somebody steps up to do that work, though, we will likely have to stick with `-O2`, even knowing that the resulting code is not as good as it could be.
