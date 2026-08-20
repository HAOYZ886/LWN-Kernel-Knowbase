---
title: Random numbers from CPU execution time jitter
url: https://lwn.net/Articles/642166/
date: "April 29, 2015"
category: Random numbers
author: "By Jake Edge April 29, 2015"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
April 29, 2015

Availability of entropy, especially for embedded devices early in the kernel boot process, is a commonly discussed problem in the kernel random number community. The usual sources of entropy depend on events like the timing of user input or disk interrupts that may not be present yet—or at all. A [patch set](<https://lwn.net/Articles/642137/>) from Stephan Müller seeks to remedy that by using entropy that is collected from a source that is always present: CPU execution time jitter. 

On a modern processor, there are many different factors that can impact the amount of time it takes to execute the same set of instructions. If one measures that execution time precisely multiple times, it will show variation—jitter. An attacker who doesn't have any special hardware-level access to the CPU cannot predict this jitter, which makes it a good source of entropy, according to Müller's lengthy [paper](<http://www.chronox.de/jent/doc/CPU-Jitter-NPTRNG.html>) on the technique. 

There are numerous CPU-level activities that lead to unpredictability in execution time. The fill level of the instruction pipelines, memory wait states, instruction and data caches, branch prediction, interrupts, power management, frequency scaling, and so on can all contribute to changing the execution time. As Müller's paper shows, a wide variety of today's processors show enough jitter (in a statistical sense) to be used as an entropy source for the kernel's random number pools. 

The heart of the algorithm to implement Müller's jitter measurement gathering lives in the `jent_measure_jitter()` function. It is effectively a ""random bit generator"", since it returns a single bit that has been calculated based on the jitter measured in that function. The first step is to introduce some noise into the measurement based on memory reads and writes. The jitter entropy module allocates a 2KB buffer during initialization that it loops through and simply adds one to the value stored there (which causes both a load and a store). The buffer is larger than the L1 cache of the processor, which should introduce some unpredictable wait states into the measurement. 

It then gets a timestamp and calculates a delta from the previous timestamp. This delta is used in a ""deliberately inefficient"" calculation to fold the value down to a single bit. There are faster ways to do the folding operation, but his algorithm is part of what is being measured for jitter in the execution time. In order to preserve those measurements, optimization for the jitter random number generator (RNG) must be turned off. In fact, there is a test in the code that will cause the build to fail if optimization is enabled to avoid the possibility of getting bad random numbers from a misconfigured build. The comment for the `jent_fold_time()` function explains a bit further: 

This function is the root cause why the code shall be compiled without optimization. This function not only acts as folding operation, but this function's execution is used to measure the CPU execution time jitter. Any change to the loop in this function implies that careful retesting must be done. 

The `jent_gen_entropy()` function generates a 64-bit random number by calling `jent_measure_jitter()` 64 times. Getting larger amounts of random data is done with `jent_read_entropy()`, which takes a buffer and length and repeatedly calls `jent_gen_entropy()` to fill the buffer with the requested amount of random data. 

Adding the [CPU jitter RNG](<http://www.chronox.de/jent.html>) is only part of what the patch set does, however. The kernel's deterministic random bit generator (DRBG) is currently initialized by making a call to `get_random_bytes()`, which uses the non-blocking random number pool. In certain circumstances (e.g. for some embedded devices or virtual machines), that pool will not have been seeded from enough events to provide the entropy required. 

In mid-April, kernel crypto subsystem maintainer Herbert Xu [asked](<https://lwn.net/Articles/642341/>) Müller whether the current DRBG implementation was compliant with the US National Institute of Standards and Technology (NIST) [SP 800-90A specification for DRBGs](<http://en.wikipedia.org/wiki/NIST_SP_800-90A>) that specifies seeding DRBGs from non-deterministic sources. Since the worst case for `get_random_bytes()` is that it is completely deterministic, Xu felt that some other mechanism should be used to seed the kernel's DRBG so that it complied. 

Müller had already [proposed](<http://thread.gmane.org/gmane.linux.kernel/1577419>) inclusion of his CPU jitter RNG in October 2013\. He used that code as the basis for this new patch set. Instead of reusing the existing blocking pool (i.e. the one that feeds `/dev/random`), though, his patch creates a new `kernel_pool` that is only accessible to the kernel itself. That eliminates a kind of denial-of-service attack where a user-space program can continuously read `/dev/random` to consume all of the entropy being generated by the system. 

The DRBG is then seeded early in the boot process from a combination of `get_random_bytes()` and the jitter RNG. In addition, an asynchronous call is made for the required amount of random data from the new, blocking `kernel_pool`. It will only return once the required amount of entropy has been gathered by the system and the random data returned will be used to reseed the DRBG. Thus, the DRBG is always seeded with non-deterministic data early on—as long as the jitter RNG is actually producing random numbers. 

Back in 2013, kernel RNG maintainer Ted Ts'o [expressed skepticism](<https://lwn.net/Articles/572242/>) about the jitter technique. He was concerned that the measurements were not as unpredictable as they appear to be—that a sufficiently knowledgeable attacker could determine enough of the state to predict the timing. 

It may be that there is some very complex state which is hidden inside the the CPU execution pipeline, the L1 cache, etc., etc. But just because *you* can't figure it out, and just because *I* can't figure it out doesn't mean that it is ipso facto something which a really bright NSA analyst working in Fort Meade can't figure out. (Or heck, a really clever Intel engineer who has full visibility into the internal design of an Intel CPU....) 

Effectively, he was worried that the entropy estimation for the jitter measurements was too high, perhaps far too high. 

Ts'o has not commented on the latest patches, at least yet. In fact, there haven't really been any technical comments on the patches as yet. Xu [seemed to indicate](<https://lwn.net/Articles/642354/>) that he is generally in favor of Müller's solution for the DRBG. If the patches do get merged, perhaps other users for the jitter RNG will emerge. It is a fairly straightforward and speedy mechanism for collecting entropy—the question is how much of that entropy is "real".
