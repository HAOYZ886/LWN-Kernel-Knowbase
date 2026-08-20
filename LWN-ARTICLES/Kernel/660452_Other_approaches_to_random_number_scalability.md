---
title: Other approaches to random number scalability
url: https://lwn.net/Articles/660452/
date: "October 21, 2015"
category: Random numbers
author: "By Jake Edge October 21, 2015"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jake Edge**  
October 21, 2015

Back in late September, we [looked](<https://lwn.net/Articles/658497/>) at a patch to improve the scalability of random number generation on Linux systems—large NUMA systems, in particular. While the proposed change solved the immediate scalability problem, there were some downsides to that approach, in terms of both complexity and security. Some more recent discussion has come up with other possibilities for solving the problem. 

The original idea came from Andi Kleen; it changed the kernel's non-blocking random number pool into a set of pools, one per NUMA node. That would prevent a spinlock on a single pool from becoming a bottleneck. But it also made the kernel's random number subsystem more complex. In addition, it spread the available entropy over all of the pools, effectively dividing the amount available to users on any given node by the number of pools. 

But, as George Spelvin [noted](<https://lwn.net/Articles/660740/>), the entropy in a pool is ""not located in any particular bit position"", but is distributed throughout the pool—entropy is a ""holographic property of the pool"", as he put it. That means that multiple readers do not need to be serialized by a spinlock as long as each gets a unique [salt](<https://en.wikipedia.org/wiki/Salt_%28cryptography%29>) value that ensures that the random numbers produced are different. Spelvin suggested using the CPU ID for the salt; each reader hashes the salt in with the pool to provide a unique random number even if the pool is in the same state for each read. 

Spelvin provided a patch using that approach along with his comments. Random number subsystem maintainer Ted Ts'o [agreed](<https://lwn.net/Articles/661422/>) with Spelvin about how the entropy is distributed, but had some different ideas on how to handle mixing the random numbers generated back into the pool. He also provided a patch and asked Kleen to benchmark his approach. ""I really hope it will be good enough, since besides using less memory, there are security advantages in not spreading the entropy across N pools."" 

Either approach would eliminate the lock contention (and cache-line bouncing of the lock), but there still may be performance penalties for sharing the pool among multiple cores due to cache coherency. The non-blocking pool changes frequently, either as data gets mixed in from the input pool (which is shared with the blocking pool) or as data that is read from the pool gets mixed back in to make it harder to predict its state. The cache lines of the pool will be bounced around between the cores, which may well be less than desirable. 

As it turned out, when Kleen ran his micro-benchmark, both patch sets [performed poorly](<https://lwn.net/Articles/661457/>) in comparison to the multi-pool approach. In fact, for reasons unknown, Spelvin's was worse than the existing implementation. 

Meanwhile, while the benchmarking was taking place, Ts'o [pointed out](<https://lwn.net/Articles/661380/>) that it may just make sense to recognize when a process is "abusing" `getrandom()` or `/dev/urandom` and to switch it to using its own cryptographic-strength random number generator (CSRNG or CRNG) seeded from the non-blocking pool. That way, uncommon—or, more likely, extremely rare—workloads won't force changes to the core of the Linux random number generator. Ts'o is hoping to not add any more complexity into the random subsystem: 

At this point, I wonder if it might not be simpler to restrict the current nonblocking pool to kernel users, and for userspace users, the first time a process reads from /dev/urandom or calls getrandom(2), we create for them a ChaCha20 CRNG, which hangs off of the task structure. This would require about 72 bytes of state per process, but normally very few processes are reading from /dev/urandom or calling getrandom(2) from userspace. 

The CRNG would be initialized from the non-blocking pool, and is reseeded after, say, 2**24 cranks or five minutes. It's essentially an OpenBSD-style arc4random in the kernel. 

Spelvin was [concerned](<https://lwn.net/Articles/661445/>) that the CSRNG solution would make long-running servers susceptible to backtracking: using the current state of the generator to determine random numbers that have been produced earlier. If backtracking protection can be discarded, there can be even simpler solutions, he said, including: ""just have *one* key for the kernel, reseeded more often, and a per-thread nonce and stream position."" But Ts'o [said](<https://lwn.net/Articles/661454/>) that anti-backtracking was not being completely abandoned, just relaxed: ""We are discarding backtracking protection between successive reads from a single process, and even there we would be reseeding every five minutes (and this could be tuned), so there is *some* anti-backtracking protection."" 

Furthermore, he suggested that perhaps real abusers could get their own CSRNG output, while non-abusers would still get output from the non-blocking pool: 

On the flip side, the time when you might care about anti-backtracking protection is say, when you're generating a new session key for a new connection. So perhaps one approach is to use some kind of ratelimiter algorithm so that if you're using /dev/urandom "carefully" (say, no more than ten times a second), we'll use the non-blocking pool. But once a process exceeds that limit, it will switch over the the CRNG, and then the only performance that abuser process will hurt is its own (because it would be even faster if they were running something like arc4random in userspace). 

Spelvin had suggested adding another random "device" (perhaps `/dev/frandom`) to provide the output of a CSRNG directly to user space, because he was concerned about changing the semantics of `/dev/urandom` and `getrandom()` by introducing the possibility of backtracking. But he agreed that changing the behavior for frequent heavy readers/callers would not change the semantics since the [random(4) man page](<http://man7.org/linux/man-pages/man4/random.4.html>) explicitly warns against that kind of usage: 

[...] so if any program reads more than 256 bits (32 bytes) from the kernel random pool per invocation, or per reasonable reseed interval (not less than one minute), that should be taken as a sign that its cryptography is not skillfully implemented. 

Spelvin posted another [patch set](<https://lwn.net/Articles/661025/>) that pursues his ideas on improving the scalability of generating random numbers. It focuses on the reducing the lock contention when the output of the pool is mixed back into the pool to thwart backtracking (known as a mixback operation). If there are multiple concurrent readers for the non-blocking pool, Spelvin's patch set ensures that one of them causes a mixback operation; others that come along while a mixback lock is held simply write their data into a global mixback buffer, which then gets incorporated into the mixback operation that is done by the lock holder when releasing the lock. 

There has been no comment on those patches so far, but one gets the sense that Ts'o (or someone) will try to route around the whole scalability problem with a separate CSRNG for abusers. That would leave the current approach intact, while still providing a scalable solution for those who are, effectively, inappropriately using the non-blocking pool. Ts'o seemed strongly in favor of that approach, so it seems likely to prevail. Kleen has [asked](<https://lwn.net/Articles/661580/>) that his multi-pool approach be merged, since ""it works and is actually scalable and does not require any new 'cryptographic research' or other risks"". But it is not clear that the complexity and (slightly) reduced security of that approach will pass muster.
