---
title: "Recent read-mostly research"
url: https://lwn.net/Articles/619355/
date: "November 11, 2014"
category: "Read-copy-update"
author: "November 11, 2014 This article was contributed by Paul McKenney"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

November 11, 2014

This article was contributed by Paul McKenney

One of my more unusual hobbies is occasionally reviewing academic papers. It should come as no surprise that I pay special attention to papers related to [read-copy update (RCU)](<https://lwn.net/Kernel/Index/#Read-copy-update>), which is a category that I am happy to see has been growing significantly over the past few years. This article gives a quick summary of some recent papers in this category. 

But first, what is RCU? It is a synchronization mechanism that was added to the Linux kernel in October 2002\. RCU is most frequently described as a replacement for reader-writer locking, but has also been used in a number of other ways. It is notable in that RCU readers do not directly synchronize with updaters, which makes RCU read paths extremely fast. It also permits readers to accomplish useful work even when running concurrently with updaters--and vice versa. 

Although the earliest known work on mechanisms resembling RCU was carried out by academics (see for example Kung and Lehman's landmark 1980 [paper](<http://dl.acm.org/citation.cfm?id=320619>)), by the early 1990s, with a few notable exceptions ([Ben Gamsa, University of Toronto, et al. [PDF]](<http://www.usenix.org/events/osdi99/full_papers/gamsa/gamsa.pdf>), [Robert Olsson, Uppsala University, et al. [PDF]](<http://www.nada.kth.se/~snilsson/publications/TRASH/trash.pdf>), [Thomas E. Hart, University of Toronto, et al. [PDF]](<http://www.rdrop.com/users/paulmck/RCU/hart_ipdps06.pdf>), and [Jonathan Appavoo, IBM T.J. Watson Research Center, et al. [PDF]](<http://www.cs.bu.edu/~jappavoo/Resources/Papers/a6-appavoo.pdf>)), much of the work in this area was carried out by practitioners. By the early 2000s, the initiative had passed to open-source projects, most notably to the Linux kernel community. 

**Quick Quiz 1** : But what can I do if I work with production-quality code but nevertheless would like to discover something new?   
Answer

However, there are welcome signs that some in academia are showing interest in RCU; see, for example, Michael L. Scott's [textbook](<http://www.morganclaypool.com/doi/abs/10.2200/S00499ED1V01Y201304CAC023>) that includes a chapter on RCU. One reason that this interest is welcome is that academics, unlike large open-source projects, are unconstrained by the need to develop production-quality code. This allows them to try crazier and more experimental ideas. 

Although many of these ideas might seem to have no value outside of academia, the fact is that many of the best yet-undiscovered ideas are protected by wide moats of insanity--with RCU itself being a case in point. Therefore, if you are unwilling to deal with insanity, the odds on your discovering something new decrease, and, of course, those of us who deal with production-quality code must be quite careful with any insanity we encounter. Furthermore, even an otherwise useless idea might inspire a developer to come up with something that is both valuable and useful, or failing that, to avoid a pitfall. Although there are no guarantees, the hope is that increased academic work in the area of read-mostly concurrency will result in new breakthroughs. 

#### Validation

Validation is one important area in which breakthroughs would be quite welcome. Peter Sewell's and Susmit Sarkar's groups at the University of Cambridge have done some [important work](<http://www.cl.cam.ac.uk/~pes20/weakmemory/>) in validating memory barriers and atomic instructions, as was [reported earlier](<https://lwn.net/Articles/470681/>). This work has been extended with greatly increased performance by some of Sewell's and Sarkar's collaborators at University College London (Jade Alglave), INRIA (Luc Maranget), and Queen Mary University of London (Michael Tautschnig), most notably in [this paper [PDF]](<http://www0.cs.ucl.ac.uk/staff/j.alglave/papers/toplas14.pdf>), which was described in this [LWN article](<https://lwn.net/Articles/608550/>). Researchers at Stony Brook University have produced an RCU-aware data-race detector, described by [Abhinav Duggal [PDF]](<http://www.filesystems.org/docs/abhinav-thesis/abhinav_thesis.pdf>) and [Justin Seyster [PDF]](<http://www.fsl.cs.sunysb.edu/docs/jseyster-dissertation/redflag.pdf>). Alexey Gotsman of IMDEA, Noam Rinetzky of Tel Aviv University, and Hongseok Yang of the University of Oxford have published [a paper [PDF]](<http://software.imdea.org/~gotsman/papers/recycling-esop13-ext.pdf>) expressing the formal semantics of RCU in terms of separation logic, and have continued with [other aspects of concurrency](<http://www.advent-project.eu>). With some luck, all of this validation work will eventually result in more and better tools for validating concurrent code. 

#### Using RCU

Phil Howard and Jon Walpole of Portland State University (PSU) have applied RCU to [red-black trees [PDF]](<http://www.usenix.org/event/hotpar11/tech/final_files/Howard.pdf>) combined with updates synchronized using software transactional memory. Josh Triplett and Jon Walpole (again of PSU) applied RCU to [resizable hash tables [PDF]](<http://www.usenix.org/event/atc11/tech/final_files/Triplett.pdf>), reported on in two parts: [Part 1](<https://lwn.net/Articles/612021/>) and [Part 2](<https://lwn.net/Articles/612100/>). (Other RCU-protected resizable hash tables have been [created by Herbert Xu](<http://thread.gmane.org/gmane.linux.network/153338>) and [by Mathieu Desnoyers](<https://lwn.net/Articles/573431/>).) 

**Quick Quiz 2** : Why don't more academic research projects make full use of open-source projects such as the Linux kernel?   
Answer

Austin Clements, Frans Kaashoek, and Nickolai Zeldovich of MIT created an [RCU-optimized balanced binary tree (Bonsai) [PDF]](<http://people.csail.mit.edu/nickolai/papers/clements-bonsai.pdf>), and applied this tree to the Linux kernel's VM subsystem ([Git trees](<http://pdos.csail.mit.edu/mosbench>)) in order to reduce read-side contention on `mmap_sem`. This work resulted in order-of-magnitude speedups and scalability up to at least 80 CPUs for a microbenchmark featuring large numbers of minor page faults. This is similar to a [patch](<https://lkml.org/lkml/2010/1/4/257>) developed earlier by Peter Zijlstra, and both were limited by the fact that, at the time, [filesystem data structures were not safe for RCU readers](<https://lkml.org/lkml/2010/1/4/532>). Clements et al. avoided this limitation by optimizing the page-fault path for anonymous pages only. More recently, filesystem data structures have been made safe for RCU readers (covered in a [2010 article](<https://lwn.net/Articles/419811/>) and [another from 2011](<https://lwn.net/Articles/452117/>)), so perhaps this work can be implemented for all page types, not just anonymous pages--Peter Zijlstra has, in fact, recently prototyped [exactly this](<https://lkml.org/lkml/2014/10/20/620>). In any case, this use of the Linux kernel itself as a testbed for cutting-edge academic research is a welcome development. 

Yandong Mao and Robert Morris of MIT and Eddie Kohler of Harvard University created another RCU-protected tree named [Masstree [PDF]](<http://www.read.seas.harvard.edu/~kohler/pubs/mao12cache.pdf>) that combines ideas from [B+ trees](<http://en.wikipedia.org/wiki/B%2B_tree>) and [tries](<http://en.wikipedia.org/wiki/Trie>). Although this tree is about 2.5x slower than an RCU-protected hash table, it supports operations on key ranges, unlike hash tables. In addition, Masstree supports efficient storage of objects with long shared key prefixes and, furthermore, provides persistence via logging to mass storage. 

The paper notes that Masstree's performance rivals that of memcached, even given that Masstree is persistently storing updates and memcached is not. The paper also compares Masstree's performance to the persistent datastores MongoDB, VoltDB, and Redis, reporting significant performance advantages for Masstree, in some cases exceeding two orders of magnitude. Another [paper [PDF]](<http://www.read.seas.harvard.edu/~kohler/pubs/tu13speedy.pdf>), by Stephen Tu, Wenting Zheng, Barbara Liskov, and Samuel Madden of MIT and Kohler, applies Masstree to an in-memory database named Silo, achieving 700K transactions per second (42M transactions per minute) on a well-known transaction-processing benchmark. Interestingly enough, Silo guarantees [linearizability](<http://en.wikipedia.org/wiki/Linearizability>) without incurring the overhead of grace periods while holding locks. 

For my part, I have been doing some work enabling complex atomic updates to RCU-protected data structures with minimal copying, which I recently [presented [PDF]](<http://www2.rdrop.com/users/paulmck/RCU/C++Updates.2014.09.11a.pdf>) at the [C++ Conference](<http://cppcon.org/>) (CppCon). (Next step: Deal with bottlenecks in user-mode memory allocators.) 

#### Using and implementing RCU

Maya Arbel and Hagit Attiya of Technion took [a more rigorous approach [PDF]](<http://www.cs.technion.ac.il/~mayaarl/podc047f.pdf>) to an RCU-protected search tree that, like Masstree, allows concurrent updates. This paper includes a proof of correctness, including proof that all operations on this tree are linearizable. Unfortunately, this implementation achieves linearizability by incurring the full latency of grace-period waits while holding locks, which degrades scalability of update-only workloads. One way around this problem is to abandon linearizability (as advocated [here [PDF]](<http://soft.vub.ac.be/races/wp-content/uploads/2012/09/p1-haas.pdf>) and [here [PDF]](<http://www.rdrop.com/users/paulmck/scalability/paper/AtomicTreeMove.2014.05.26a.pdf>)), however, Arbel and Attiya instead created an RCU variant that reduces low-end grace-period latency. Of course, nothing comes for free, and this RCU variant appears to hit a scalability limit at about 32 CPUs. Although I personally favor dropping linearizability, thus gaining _both_ performance and scalability, it is very good to see academics experimenting with alternative RCU implementations. Researchers at Charles University in Prague have also been working on RCU implementations, including dissertations by [Andrej Podzimek [PDF]](<https://andrej.podzimek.org/thesis.pdf>) and [Adam Hraska [PDF]](<http://www.helenos.org/doc/theses/ah-thesis.pdf>). 

RCU-like mechanisms are also finding their way into Java. [Sivaramakrishnan et al.](<http://doi.acm.org/10.1145/2258996.2259005>) use an RCU-like mechanism to eliminate the read barriers that are otherwise required when interacting with Java's garbage collector, resulting in significant performance improvements. 

#### A specialized RCU implementation and its use

The final piece of academic work that is described in this article was carried out at the Shanghai Jiao Tong University by Ran Liu, Heng Zhang, and Haibo Chen, who created a specialized variant of RCU that they used for an optimized reader-writer lock. This is important because traditional reader-writer lock implementations, such as the Linux kernel's `rwlock_t`, do not scale well. The reason for this is that all readers must atomically update the single memory location that represents the `rwlock_t`, both with entering and when leaving their read-side critical sections. The cache line containing this memory location will shuttle among all of these CPUs. Each access to this memory location will therefore result in a cache miss, which will in turn result in the accessing CPU stalling for hundreds or even thousands of clock cycles. 

The more CPUs attempting to acquire or release the lock, the longer the stalls. Unless the read-side critical sections are extremely long (hundreds of thousands or millions of instructions), the result will be poor performance and scalability. Furthermore, some of the Linux kernel's lock implementations favor readers over writers (which can be OK), but to the extent of starving waiting writers (which can be a big problem). 

This is, of course, why RCU is often used, but RCU has different semantics than do reader-writer locks. In a surprisingly large number of situations, this difference in semantics is not a problem. However, there are some parts of the Linux kernel that are notoriously difficult to apply RCU to. This situation has motivated a number of attempts to find more scalable implementations of reader-writer locking, going all the way back to `brlock` in the 2.4 Linux kernel (which has since been replaced by `[lglocks](<https://lwn.net/Articles/401738/>)`). These implementations provide a lock for each CPU. To read-acquire a lock, a CPU acquires its own lock, which still requires an atomic instruction and memory barrier on most architectures, but does _not_ incur a cache miss in the absence of writers. Writers must acquire all CPUs' locks, which requires an atomic instruction and usually a cache miss per CPU. This overhead increases linearly with the number of CPUs, which can result in high overhead for updates. 

In some cases, decreased read-side overhead is required. One [scheme](<http://lkml.org/lkml/2006/10/26/73>) was put forward by Gautham Shenoy, and a similar [approach](<https://patchwork.kernel.org/patch/2157401>) was put forward by Srivatsa Bhat. The idea behind both of these patches is that readers check a flag before entering their read-side critical sections. If that flag indicates that there are no writers, then the readers proceed without executing any atomic instructions or memory barriers, and, in the common case, without incurring any cache misses. However, nothing comes for free, and for these approaches, the penalty is paid by the writers. 

A writer must first set the flag, then wait for all the pre-existing readers, who might have loaded the flag before the writer set it. This is handled by requiring readers to check the flag within an RCU read-side critical section, and, if the flag is clear, to execute their entire read-side critical section under RCU protection. This allows the writer to use `synchronize_rcu()` after setting the flag to wait for pre-existing readers. Of course, the resulting grace-period latency can be problematic in many cases. This latency can be reduced by using expedited grace-period primitives such as `synchronize_rcu_expedited()`, but these result in high CPU overhead as well as a set of inter-processor interrupts (IPIs) to other CPUs. The high overhead can reduce throughput, and the IPIs can spell trouble for real-time applications. 

Liu, Zhang, and Chen take a slightly different approach in their [USENIX paper](<http://blogs.usenix.org/conference/atc14/technical-sessions/presentation/liu>). As noted earlier, they use a special-purpose RCU implementation tailored specifically for reader-writer locking, which they call a "passive reader-writer lock" or prwlock. In a manner roughly similar to [signal-based user-space RCU](<http://urcu.so>), writers increment a version number, and then wait for all CPUs to acknowledge the new version. Readers automatically acknowledge when acquiring or releasing the lock, but CPUs that are not using the lock will not acknowledge the new version. 

One way of handling this would be to place full memory barriers in the read-side code, but this group decided to strive for ultimate read-side performance, which means that the read-to-write transition involves IPIs. The idea is that if a given CPU has not either entered or exited a read-side critical section in a timely fashion, it is sent an IPI, which causes it to respond. The group also looked at a few optimizations, including scoping the IPIs on a per-process basis for `mmap_sem`, optimizing the wakeup path, and, for user-space implementations, using an in-kernel helper similar in some ways to [`sys_membarrier()`](<https://lwn.net/Articles/369567/>). This optimized reader-writer lock provided significant increases in performance for systems of up to 64 CPUs on a number of workloads (see pages 11-13 of the paper for details). 

Like most conferences, USENIX imposes strict length limits, which means that this paper does leave some unanswered questions. 

One question is whether the number of IPIs could be reduced by taking advantage of the Linux kernel's per-CPU dyntick-idle state. This seems possible, and it seems especially beneficial for [`NO_HZ_FULL`](<https://lwn.net/Articles/549580/>) kernels, where it would prevent IPIing time-critical user-mode realtime and HPC applications. It would be interesting to see how this approach plays out. 

A second question involves Figure 17 on page 12, which shows that an RCU-based resizable hash table takes considerably longer to resize than does one protected by the prwlock. This is quite true, but it is also true that when using RCU, reads and updates can proceed concurrently with the resizing. It would be interesting to directly measure the actual effect of a resize operation on read and update throughput and latency. 

A third question involves the paper's comparison of a user-mode implementation of the reader-writer lock to signal-based user-space RCU. In this case, the reader-writer lock used an in-kernel optimization. However, user-space RCU also has an in-kernel optimization in the form of `sys_membarrier()`. It would therefore be interesting to compare both mechanisms using their in-kernel optimizations. Similarly, it would be interesting to compare prwlock to RCU expedited grace periods as well as the slower normal grace periods. 

A fourth question involves configuration of the benchmark runs. For example, the comparisons of prwlock to user-space RCU set user-space RCU's batch size to one. This is of course an attractive setting if you wish to highlight prwlock, but it would be more interesting to compare against the much larger default setting of 4096 for the batch size. It also raises the question of how and to what extent prwlock can amortize single sets of IPIs over the write acquisitions of multiple prwlocks. 

**Quick Quiz 3** : Given the need to examine per-CPU data for each and every CPU, isn't prwlock inherently non-scalable?   
Answer

A final question involves scalability, both in terms of numbers of prwlocks and in terms of system size. Exploring these areas should provide much interesting future work. All these questions aside, and in part because of these questions, this was an extremely interesting paper that might well have significant practical applicability. 

#### Conclusions

It is very good to see increased academic interest in read-mostly techniques such as RCU. We can all look forward to much interesting (and hopefully some useful) work along the way. 

#### Acknowledgments

We all owe thanks to Davidlohr Bueso, Austin Clements, Hagit Attiya, Maya Arbel, Eddie Kohler, and Haibo Chen for their help making this human-readable. I am grateful to Jim Wasko for his support of this effort. 

####  Answers to Quick Quizzes

**Quick Quiz 1** : But what can I do if I work with production-quality code but nevertheless would like to discover something new? 

**Answer** : There are no guarantees. That said, here are some things you can try: 

  1. Set up automated testing for your code, and make this testing as vicious as you possibly can. Otherwise, you will be spending all your time dealing with bugs and will therefore have no time to discover something new. (Yes, it is quite possible that one of your bugs will lead to something new, but the odds are stacked against you.) 
  2. Learn as much as you can about the area of your interest. However, emphasize learning by doing, and especially try doing things wrong to see what happens. 
  3. Study things completely unrelated to your area of interest. It is surprising how often standard practice in one area inspires something new in another area. 
  4. Seek out trouble, especially when the trouble involves doing something that has never been done before. (Important safety tip: It is almost as easy to overdo this particular piece of advice as it is to avoid it completely.) 
  5. Make heavy use of source-code control systems such as Git in order to manage the resulting chaos--or at least to keep the chaos safely removed from your production systems. 

With some luck, persistence, and hard work, who knows what you might find? 

**Back to Quick Quiz 1**.

**Quick Quiz 2** : Why don't more academic research projects make full use of open-source projects such as the Linux kernel? 

**Answer** : As you can see from this article, a goodly number of researchers do use Linux. That said, the size and complexity of the Linux kernel can be problematic for some research projects--considerable courage and talent are required to take on Linux-kernel internals. 

**Back to Quick Quiz 2**.

**Quick Quiz 3** : Given the need to examine per-CPU data for each and every CPU, isn't prwlock inherently non-scalable? 

**Answer** : Not necessarily. 

For example, there is no reason why multiple kernel threads could not be brought to bear in order to parallelize the scan of the per-CPU data. In fact, if examining one CPU's data required 100 ns and waking up a helper kthread required 5 us, then it might make sense to provision a helper kthread for each group of (say) 100 CPUs. 

That said, this approach would get quite "interesting" from irq-disabled regions of code. 

**Back to Quick Quiz 3**.
