---
title: Adding CPU randomness to the entropy pool
url: https://lwn.net/Articles/586427/
date: "February 19, 2014"
category: "Random numbers; Security-Random number generation"
author: "By Jake Edge February 19, 2014"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jake Edge**  
February 19, 2014

Kernel developers, or at least the maintainers of the random subsystem, are always on the lookout for sources of unpredictability for use in the random number pool. In particular, there are a number of scenarios where good random numbers are needed and the available pool is lacking in quality—embedded systems, early in the boot process, virtual machines, etc.—so new sources that can alleviate the problem are generally welcome. However, there is always the question of how much entropy is truly provided by these sources, which is a difficult problem to solve. Two recent patches would contribute unpredictable sources, but take a different approaches with regard to adding to the store of entropy. 

The kernel has two separate interfaces for random number generation: `/dev/random` and `/dev/urandom`. They are supposed to be used for different purposes, with `/dev/random` only providing as many bits of randomness as bits of entropy have been added to the pool—blocking if insufficient entropy is available. It is meant to be used for long-lived keys (e.g. the SSH key for the system), while `/dev/urandom` provides cryptographic-strength pseudo-random numbers without the entropy requirement (and, thus, no blocking). Data read from either device comes from the same pool, but the entropy requirement is only applied for reads from `/dev/random`. 

Unpredictable events measured by the kernel (that cannot be observed by an adversary) make up the input to the entropy pool from which the random numbers are generated. Various kinds of interrupts are used (e.g. intra-key timing from the keyboard, sometimes disk or network device intra-interrupt timing, and so on) and their values are mixed into the pool. As that is done, an estimate of how many bits of entropy are being contributed is added to the entropy count. That estimate is hopefully conservative enough that it underestimates the amount of true entropy in the pool, while trying not to make it impossible to generate a reasonable number of random bits in a reasonable time. 

An even more conservative approach would be to add unpredictable data to the pool without crediting any entropy. That is already done with some sources in today's kernel, such as when adding unpredictable device-specific data using `add_device_randomness()`. There is value in adding "zero credit" (i.e. no entropy contributed) unpredictability to the pool. Any data that is added perturbs the state of the pool, which will change the values produced by `/dev/urandom`. In some situations, the same random numbers would be produced boot after boot if there were no unpredictable data added. 

#### CPU randomness

"Zero credit" is the approach Jörn Engel took with his [CPU randomness patch](<https://lwn.net/Articles/584005/>). It mixes uninitialized stack values with unpredictable values like `jiffies` into its own pool, then mixes that pool into the normal entropy pool periodically. It clearly adds unpredictability into the pool, but how much entropy it provides is hard or impossible to determine, so Engel gives it no entropy credit. 

The patch gathers its info from two kernel locations: the scheduler and the slab allocator. It uses an uninitialized four-element array (`input`) on the stack and XORs various values into it to produce the input to the private pool. The values used are `jiffies`, the value of the time stamp counter (TSC), the address where the scheduler and allocator functions will return, a value specific to that invocation of the scheduler or allocator, and the address of the `input` array itself. It is similar in some ways to the gathering that is done for interrupts for the global pool. This collection and mixing is done quite frequently (whenever `need_resched()` or `__do_kmalloc()` are called), then the private pool is combined with normal pool at most once per second. 

Perhaps surprisingly, Engel's tests showed no measurable impact on kernel performance. For the private pool, he is using a custom mixing algorithm that is faster than `fast_mix()` that is used on the global pool, but provides worse mixing. The normal path is used when mixing the private pool into the global. 

Engel's [focus](<https://lwn.net/Articles/587006/>) is on ""generating high-quality randomness as soon as possible and with low cost to the system"". In particular, he is targeting embedded systems: 

But on embedded systems with less modern CPUs, few interrupt sources, no user interface, etc. you may have trouble collecting enough randomness or doing it soon enough. That is the problem worth fixing. 

While the values being used _seem_ unpredictable, Ted Ts'o [questioned](<https://lwn.net/Articles/587013/>) whether an ""attacker with deep knowledge of how the kernel was compiled and what memory allocations get done during the boot sequence"" would be able to successfully predict some of the values. For many kernel deployments (e.g. distribution kernels), an attacker will be able to get that deep knowledge fairly easily. Ts'o thought Engel's approach had promise for improving the `/dev/urandom` output, but agreed with the approach of not crediting entropy (thus not affecting how much data is available from `/dev/random`). 

#### CPU jitter

Another approach was suggested by Stephan Müller in his [CPU Jitter random number generator (RNG) patch set](<https://lwn.net/Articles/584105/>). It was met with more skepticism, at least partly because it _does_ add to the entropy count. Ts'o and others are not convinced that sufficiently knowledgeable attackers couldn't predict the output. Müller's reliance on statistical techniques in his [paper](<http://www.chronox.de/jent/doc/CPU-Jitter-NPTRNG.html>) to measure the entropy pool and RNG output is also a cause for some skepticism. But, [according to Müller](<https://lwn.net/Articles/587026/>), the statistical measures are just a ""necessary baseline"" before he gets into ""measuring the actual noise coming out of the noise sources"". 

Müller's method is to measure the jitter in the amount of time it takes the CPU to perform a set of operations. When entropy is needed, the driver repeatedly runs two "noise sources": a memory accessing routine that ""will add to the timing variations due to an unknown amount of CPU wait states added when accessing memory"" and a timestamp folding operation that is ""deliberately inefficient"", which requires the function to be built with no optimization (`-O0`). The folding operation turns a 64-bit timestamp into one bit that is XORed into the driver's entropy pool. The jitter in the timing of those two operations is also mixed into that entropy pool one bit at a time. Once the required entropy is available, random numbers derived from that are returned. 

While the timing is unpredictable, due to a number of the factors Müller cites in his paper and patchset, it still amounts to a pseudo-random number generator (PRNG), [according to H. Peter Anvin](<https://lwn.net/Articles/587020/>): 

The more modern and high performance a design you have the more sources of unpredictability there are. However, there are very few, if any, (unintentional) sources of actual quantum noise in a synchronous CPU, which means that this is at its core a PRNG albeit with a large and rather obfuscated state space. 

He goes on to say that independent clocks in a system would provide a source of quantum noise that could potentially be used to increase the entropy count, but that such clocks are rare on today's systems as clocks are typically slaved from the same source using phase-locked loops to keep them synchronized. Thus, using jitter (or Engel's CPU randomness) for mixing into the pool is reasonable, Anvin continued, but not for entropy credit: 

As mentioned, I definitely have no objection to these sort of things as zero-credit entropy sources -- they cannot, by definition, do harm, unless they somehow cancel other inputs out -- but the notion of making them creditable sources makes me skeptical in the extreme. 

It would be nice to assume that since there is no discernible pattern to the output, there must be an underlying entropy-adding event at play. But that is not enough for Ts'o, Anvin, and others to be convinced. Back in October, when the CPU Jitter RNG was first introduced, Ts'o [replied at length](<https://lwn.net/Articles/572242/>) to the patch and explained the problem he saw: 

It may be that there is some very complex state which is hidden inside the the CPU execution pipeline, the L1 cache, etc., etc. But just because *you* can't figure it out, and just because *I* can't figure it out doesn't mean that it is ipso facto something which a really bright NSA analyst working in Fort Meade can't figure out. (Or heck, a really clever Intel engineer who has full visibility into the internal design of an Intel CPU....) 

He also went on to describe ways that Müller could convince him that there is real random noise being generated: 

So if you want to really convince the world that CPU jitter is random, it's not enough to claim that it you can't see a pattern. What you need to do is to remove all possible sources of the uncertainty, and show that there is still no [discernible] pattern after you do things like (a) run in kernel space, on an otherwise [quiescent] computer, (b) disable interrupts, so that any uncertainty can't be coming from interrupts, etc., Try to rule it all out, and then see if you still get uncertainty. 

If you think it is from DRAM timing, first try accessing the same memory location in kernel code with the interrupts off, over and over again, so that the memory is pinned into L1 cache. You should be able to get consistent results. If you can, then if you then try to read from DRAM with the L1 and L2 caches disabled, and with interrupts turned off, etc, and see if you get consistent results or inconsistent results. If you get consistent results in both cases, then your hypothesis is disproven. If you get consistent results with the memory pinned in L1 cache, and inconsistent results when the L1 and L2 cache are disabled, then maybe the timing of DRAM reads really are introducing entropy. But the point is you need to test each part of the system in isolation, so you can point at a specific part of the system and say, *that*'s where at least some uncertainty which an adversary can not reverse engineer, and here is the physical process from which the [chaotic] air patterns, or quantum effects, etc., which is hypothesized to cause the uncertainty. 

Müller has done most or all of the testing Ts'o suggested as reported in his paper. The results seem to bear out some kind of random noise in both the memory access and folding operations. But Anvin's opinion that the jitter in modern CPUs just represents a complicated PRNG space seems to have held the day. Perhaps a further look at the testing results is in order. 

The reliance of the jitter RNG on a high-resolution timer makes it unsuitable for Engel's embedded use case (as some of those systems lack such a timer), so it's not at all clear where things go from here. Ts'o is not opposed to adding something as a zero-entropy source to try to get better `/dev/urandom` numbers earlier in the boot. Since Engel's solution is both simpler and does not rely on a high-resolution timer, it may well get the nod.
