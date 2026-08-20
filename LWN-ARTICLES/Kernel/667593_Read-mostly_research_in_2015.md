---
title: "Read-mostly research in 2015"
url: https://lwn.net/Articles/667593/
date: "December 16, 2015"
category: "Read-copy-update"
author: "December 16, 2015 This article was contributed by Paul McKenney"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

December 16, 2015

This article was contributed by Paul McKenney

[_Editor's note: this article was co-written by Paul McKenney and Aravinda Prasad_]. 

It has been just over one year since the last LWN article on [read-mostly research](<http://lwn.net/Articles/619355/>). However, it is good to see that there have been a number of interesting papers since then related to RCU and other read-mostly topics. This article gives a quick summary of them. 

#### Validation

Joseph Tassarotti (Carnegie-Mellon University), Derek Dreyer (Max Planck Institute for Software Systems), and Viktor Vafeiadis (also MPI-SWS) carried out a manual proof of correctness of the quiescent-state-based reclamation (QSBR) variant of [user-space RCU](<http://liburcu.org>). This paper modeled `rcu_dereference()` with C11 `memory_order_acquire` loads rather than the `memory_order_consume` loads intended for that purpose. However, given [the current parlous state of `memory_order_consume` [PDF]](<http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/p0098r0.pdf>), this shortcut is quite understandable. Their paper is titled "[Verifying Read-Copy-Update in a Logic for Weak Memory [PDF]](<http://www.mpi-sws.org/~dreyer/papers/rcu/paper.pdf>)" and appeared in the 2015 Proceedings of the 36th annual ACM SIGPLAN conference on Programming Language Design and Implementation. Another researcher asked me if I felt that their assumptions were adequate, and as previously reported [here](<http://paulmck.livejournal.com/40129.html>), I replied that, since they found no bugs, their assumptions clearly must be unrealistic. In the authors' defense, the proof is highly non-trivial, so the lack of bugs was not due to lack of effort. 

Iftekhar Ahmed of Oregon State University gave [a presentation](<https://linuxplumbersconf.org/2015/ocw/proposals/2751>) to the Scalability microconference of the [2015 Linux Plumbers Conference](<https://linuxplumbersconf.org/2015/>) describing a "mutant" technique for verifying test suites that he is applying to Linux-kernel RCU. Each mutant is a copy of the RCU code, but with a single change, or "mutation". For example, a given statement might be deleted, a comparison operator might be changed, or a constant might be changed. The question is then whether `rcutorture` will complain about the mutant. Mutants that `rcutorture` does not complain about are said to have "survived". 

Of course, some mutants will never produce a complaint. For example, a mutation that removes a `BUG_ON()` or that changes (say) "`while (1)`" to "`while (2)`" produces a correct program. Some manual inspection of surviving mutants is therefore required. However, such manual inspection has been rewarded by the finding of not one but two holes in `rcutorture`, one of which was hiding a real bug--in Tiny RCU, believe it or not. [Full disclosure: I am collaborating with Iftekhar and his professors, Alex Groce and Carlos Jensen, and am co-author on a resulting [paper [PDF]](<http://www.cs.cmu.edu/~agroce/ase15.pdf>).] 

**Quick Quiz 1** : Wait a minute!!! How can you possibly create a logic expression that represents all executions of a parallel program???   
Answer

The past year also saw the release of version 5 of the University of Oxford C Bounded Model Checker ([CBMC](<http://www.cprover.org/cbmc/>)), led by Daniel Kröning. CBMC takes a C program as input, and creates a logic expression that takes the program's inputs as inputs, and that evaluates to `true` if some combination of those inputs can trigger an assert or result in an array-out-of-bounds condition. This new version introduces multiprocessor capability (including some weak-memory support), and is capable of automatically verifying some trivial RCU implementations, including the Linux kernel's [Tiny RCU](<http://paulmck.livejournal.com/39343.html>). It is early days for CBMC's multiprocessor feature, but its appearance is a welcome development. 

#### Using and implementing RCU

Mike Ash [posted](<https://www.mikeash.com/pyblog/friday-qa-2015-05-29-concurrent-memory-deallocation-in-the-objective-c-runtime.html>) a description of an RCU-like primitive in Apple's Objective-C runtime. Interestingly enough, this approach identifies read-side critical sections via designated code ranges. This of course means that it scans all CPUs' program counters in order to identify grace periods. This approach is interesting in being a distinct method of achieving zero-overhead read-side critical sections, albeit one that poses some interesting practical challenges for large read-side critical sections that call many functions and for reliable sampling of other CPUs' program counters without undue degradation of realtime response. 

**Editor's note** : for those wondering how an algorithm that takes locks can be "lock-free," you're not alone. According to Paul, in [common academic usage](<https://en.wikipedia.org/wiki/Non-blocking_algorithm>), "lock-free" means "at least one thread is guaranteed to make progress." Algorithms that do not take locks at all, instead, are "lockless." 

Pedro Ramalhete and Andreia Correia took a much simpler approach, using reader-writer locking to implement RCU, resulting in what they call [poor-man's URCU [PDF]](<https://github.com/pramalhe/ConcurrencyFreaks/blob/master/papers/poormanurcu-2015.pdf>). Although this is by no means the first lock-based implementation of RCU, it is, as far as I know, the first lock-based implementation that boasts lock-free readers. Their trick is to use not one but two reader-writer locks. Readers loop doing a read-side trylock on each of these two locks in turn, exiting the loop upon successful acquisition of either of the two locks. Writers are serialized by another lock, and simply write-acquire and write-release the two reader-writer locks in succession. If a writer is delayed while write-holding one of these two locks, readers can still make progress by acquiring the other lock. Although poor man's RCU seems unlikely to become a production-quality implementation of RCU, it is worth studying in its own right. After all, it is not every day that someone achieves non-blocking forward-progress guarantees with what is usually considered to be a blocking primitive! 

Maya Arbel and Adam Morrison, both of Technion, wrote a paper titled "[Predicate RCU: An RCU for Scalable Concurrent Updates [PDF]](<http://www.cs.technion.ac.il/~mad/publications/ppopp2015-prcu.pdf>)". This title might come as quite a surprise to those of us for whom RCU has long provided eminently scalable updates. However, Arbel and Morrison were working with an internal tree whose non-leaf nodes can contain data, described [here [PDF]](<https://pdfs.semanticscholar.org/73e4/cd29273cf9d98d35bc184330e694ba798987.pdf>). It turns out that deleting an item from an internal tree is more complex than in an external tree where only leaf nodes contain data. This added complexity is especially vexing because RCU-protected readers cannot be excluded, which is a particular problem when those readers are finding the spot in which to do an insertion. This interaction between complex deletions and RCU lookup-insertions is handled by holding locks across grace periods, which can degrade both performance and scalability. Of course, holding locks across grace periods places those grace periods on the critical path, which motivated the authors to work hard to reduce grace-period duration, hence predicate RCU. 

Predicate RCU allows updaters to wait only on those readers that are involved with the update in question, which should shorten grace periods. This should in turn reduce the penalty for holding locks across grace periods, however, there is no free lunch: Shorter grace periods mean fewer updates per grace period and thus higher per-update overhead. This effect is anything but subtle: a [2004 USENIX paper](<https://www.usenix.org/conference/2004-usenix-annual-technical-conference/making-rcu-safe-deep-sub-millisecond-response>) notes that RCU has been observed satisfying more than 1,000 updates with a single grace period while running an unremarkable workload. Nevertheless, if you must hold a lock across a grace period, a shorter grace period is going to be a good thing, as can be seen in the Linux kernel's `synchronize_net()` function, which uses `synchronize_rcu_expedited()` when called with the networking layer's RTNL lock held. 

**Quick Quiz 2** : Given the performance penalties, shouldn't someone stop researchers from holding locks across grace periods?   
Answer

Therefore, given that Arbel and Morrison need to hold locks across RCU grace periods, they need short grace periods. To this end, predicate-RCU readers associate themselves with an algorithm-specific value. For example, readers of a hash table might associate themselves with the lookup key, which would allow updaters to wait only for those readers whose keys hashed to the hash bucket being updated. This is in some ways similar to the Linux kernel's [SRCU](<http://lwn.net/Articles/202847/>), where the `srcu_struct` serves as the value associating readers and updaters. Either way, given that updaters only need to wait on a small subset of the readers, one would expect grace periods to elapse more quickly, which would in turn be expected to reduce the penalty for waiting for a grace period while holding a lock. Their performance results meet this expectation: Although they do not achieve linear scalability, shorter grace periods do improve their performance. Of course, avoiding waiting for grace periods while holding locks would likely improve performance and scalability even more! 

Developers who assume that academics ignore their work will be happy to see that this paper cites a couple of LWN articles. 

Yujie Liu (Lehigh University), Victor Luchangco (Oracle Labs), and Michael Spear (also Lehigh) wrote a 2013 paper titled "[Mindicators: A Scalable Approach to Quiescence [PDF]](<http://www.cse.lehigh.edu/~spear/liu_icdcs_2013.pdf>)". This paper presses the [scalable non-zero indicator (SNZI) [$PDF]](<http://dl.acm.org/citation.cfm?id=1281106&dl=ACM&coll=DL&CFID=724480715&CFTOKEN=99444769>) technique into service as a grace-period mechanism, and compares it to several other approaches. The intent appears to be to use this mechanism to implement [transactional memory (TM)](<https://en.wikipedia.org/wiki/Transactional_memory>), which also appears to require low-latency grace periods, in part courtesy of TM's linearizability requirements, which in turn seems to limit scalability. They do call out the relationship to RCU. 

Alexander Matveev (MIT), Nir Shavit (MIT and Tel-Aviv University), Pascal Felber (University of Neuchatel), and Patrick Marlier (also University of Neuchatel) recently published a [Symposium on Operating Systems Principles](<http://www.ssrc.ucsc.edu/sosp15/index.html>) (SOSP) paper titled "[Read-Log-Update: A Lightweight Synchronization Mechanism for Concurrent Programming [PDF]](<http://sigops.org/sosp/sosp15/current/2015-Monterey/printable/077-matveev.pdf>)", which can be thought of as a software transactional memory (STM) extension that includes explicitly marked read-only transactions. However, unlike RCU readers, these read-only transactions are guaranteed to see a point-in-time snapshot of the union of all RLU-protected data structures across multiple traversals. Of course, this does impose significant additional overhead on RLU updaters, as they acknowledge in their Figure 7, which shows RLU updaters being 2-5 times slower than RCU updaters. That said, this figure shows a benchmark that favors RCU rather heavily. With or without the point-in-time snapshots, I believe that their realization of the importance of explicitly marking read-only operations is a great step forward. Updates to shared variables are also explicitly marked, providing performance benefits over pure STM similar to those of [SwissTM [PDF]](<http://infoscience.epfl.ch/record/136702/files/pldi127-dragojevic.pdf>). 

**Quick Quiz 3** : If RCU does not provide readers a guaranteed consistent snapshot of the data structure, how can anyone successfully use it?   
Answer

Their technique scales reasonably well up to 16 CPUs, however, this is a very small system for modern non-mobile workloads. They do have one graph (uppermost graph in Figure 8) that goes up to 80 CPUs, but this shows poor scalability. My first thought was that this poor scalability was due to their single global counter that is atomically incremented on each update, but this did not make sense because RLU is outperforming RCU. Instead, the culprit seems to be the need to hold locks across grace periods. Because RCU optimizes for low per-update overhead at the expense of grace-period latency, and because `synchronize_rcu()` was used (instead of `call_rcu()` or `synchronize_rcu_expedited()`), holding locks across grace periods hurts RCU even more than it hurts RLU. Once again, I recommend either releasing locks before waiting for grace periods or using the asynchronous `call_rcu()` primitive: Both approaches avoid degraded scalability. In (thankfully rare) Linux-kernel cases where it is absolutely necessary to wait for grace periods while holding a mutex, the `synchronize_rcu_expedited()` APIs can be used, though these are not particularly good for realtime applications (with the exception of `synchronize_srcu_expedited()`). 

Their performance testing includes both user-space and Linux-kernel scenarios. In the Linux-kernel scenarios shown in Figure 9, their list-traversal code beat that of the Linux kernel by a surprisingly large margin. In an impressive display of good sportsmanship, one of the authors (Marlier) located the Linux-kernel performance bottleneck and [submitted a fix](<http://git.kernel.org/linus/8db70b132dd57696cfc7560203a72e90c51bfdda>) that causes the Linux kernel's lists to outperform those of the paper. The problem was a single non-atomic store and load to an unshared location in the running task's stack, with no memory barriers. It appears that current microprocessors' pipelines can be a bit slow to handle a load from a location that the current CPU just stored to. Patrick eliminated this store and load, and his patch was accepted during the v4.4 merge window. 

As far as I know, this is the first academic use of the `rcutorture` test suite to test an alternative RCU-like implementation, which is a nice milestone. Those wanting some more detailed discussion on RLU, including graphics showing scalability issues on larger systems, can find it [on this page](<https://lwn.net/Articles/667720/>). 

At the end of the paper, the authors express hope that RLU will be used both in kernel and in user-space programs. This of course raises the question of what situations would be best suited for an RLU solution. Two possibilities come to mind: 

**Quick Quiz 4** : Shouldn't RLU be tried on complex RCU uses such as the Linux kernel's VFS dentry cache-walk code?   
Answer

  * Situations where RCU readers are used in conjunction with sequence locking. These situations are already paying the complexity and performance costs of retries, so these RLU disadvantages might be less of a problem in this case. 

  * Situations where a problematic reader-writer lock has proven difficult to convert to RCU. Perhaps RLU's snapshotted readers might better handle some of these situations. 

That said, I strongly suspect that successful application of RLU will require the authors to carefully separate those semantics that are actually required from those semantics that are merely fashionable. 

It is easy to get irritated at academics' insistence on linearizability, given the large performance and scalability penalties they pay to achieve it. On the other hand, they do use deferral to improve performance of RLU, and perhaps further work along these lines will persuade them to let go of linearizability. 

Frans Kaashoek presented on the [history of parallelism and operating systems [PDF]](<http://sigops.org/sosp/sosp15/history/08-kaashoek-slides.pdf>) during the [SOSP 2015 History Day Workshop](<http://sigops.org/sosp/sosp15/history/index.html>) to mark SOSP's 50th anniversary. Frans devoted a full slide of a 33-slide deck to RCU, which should be a point of pride for the many developers who have applied RCU within the Linux kernel. (Yes, I am happy on behalf of my work on the RCU infrastructure, but let's face it, the infrastructure is profoundly uninteresting without its many uses.) 

Peter Denning also presented in SOSP 2015 History Day, but on [OS Foundations [PDF]](<http://sigops.org/sosp/sosp15/history/01-denning-slides.pdf>). Those who know me will not be surprised to hear that Peter's last bullet on his slide 39 resonated with me: "Theory follows practice". 

So what have I been doing? For my part, I have continued my work enabling complex atomic updates to RCU-protected data structures with minimal copying, which I recently [presented [PDF]](<http://www.rdrop.com/users/paulmck/scalability/paper/Updates.2015.01.16b.LCA.pdf>) at [linux.conf.au](<http://linux.conf.au/>). This round dealt with some bottlenecks in user-mode memory allocators, relearning the lesson that Glibc `malloc()` is not at all scalable. I have also been working to repair C11's and C++11's `memory_order_consume` feature, which just might be nearly done. I rewrote the Linux kernel's `synchronize_rcu_expedited()` and `synchronize_sched_expedited()` to reduce OS jitter and, for good measure, added kernel parameters to suppress expedited grace periods entirely. I also started work on design documentation for the Linux kernel's RCU implementation, starting with [its requirements](<https://lwn.net/Articles/652156/>). Finally, I have upgraded rcutorture in an attempt to keep up with the Linux kernel's huge installed base. 

#### Conclusions

It is very good to see continued academic interest in read-mostly techniques such as RCU. I very much hope that the mutual learning process continues, and that it benefits Linux developers and their users! 

#### Acknowledgments

We all owe thanks to Srivatsa Bhat, K. Gopinath, Orran Krieger, Pedro Ramalhete, Mike Ash, Derek Dreyer, Joe Tassarotti, Iftekhar Ahmed, Adam Morrison, and Hagit Attiya for helping to make this human-readable. I am grateful to Jim Wasko for his support of this effort. 

####  Answers to Quick Quizzes

**Quick Quiz 1** : Wait a minute!!! How can you possibly create a logic expression that represents all executions of a parallel program? 

**Answer** : CBMC converts the parallel program into [single static assignment (SSA)](<https://en.wikipedia.org/wiki/Static_single_assignment_form>) form, which results in each variable having a separate instance for each assignment to it. Each read from that variable is then wired up to one of the instances that it might have read its value from, and all combinations are represented in the resulting logic expression. 

Of course the resulting logic expression might be quite large, but modern computers and modern [SAT solvers](<https://en.wikipedia.org/wiki/Boolean_satisfiability_problem>) have grown quite capable. For example, in 1990, a world-class SAT solver might be able to handle 100 variables, that is, three 32-bit variables with four bits left over. In contrast, in 2015, I have solved a 1.8 million variable SAT problem on my laptop. 

The solution took a full ten seconds. 

**Back to Quick Quiz 1**.

**Quick Quiz 2** : Given the performance penalties, shouldn't someone stop researchers from holding locks across grace periods? 

**Answer** : No. 

First, please note that holding locks (or, in the Linux kernel, mutexes) across grace periods makes perfect sense in some cases, for example, in the `synchronize_net()` example given earlier. In addition, in some cases, holding locks across grace periods on rarely executed slow paths can greatly reduce complexity. Second, even though I recommend strongly against holding locks across grace periods on fast paths, it is quite possible that researchers exploring this technique will nevertheless come up with something useful. That said, I do indeed suspect that they would make better progress if they moved their grace periods outside of locks. Third, waiting for highly optimized grace periods is not likely to be a big problem on small systems. (Pretty amazing that a 64-CPU system can now be considered "small", isn't it?) Fourth and finally, it is their job to figure out what they work on. I would no more wish to dictate what they do than I would wish them to dictate what I do. 

**Back to Quick Quiz 2**.

**Quick Quiz 3** : If RCU does not provide readers a guaranteed consistent snapshot of the data structure, how can anyone successfully use it??? 

**Answer** : It turns out that RCU does in fact guarantee a consistent snapshot--at zero cost--in some important special cases. Perhaps the most common case is a data structure where updaters are restricted to adding new elements or removing old ones, in other words, where in-place updates are prohibited. If each reader only ever looks up a single element, then readers will automatically see a consistent snapshot of that single element--even if a given reader looks up a different element each time it executes. 

There are many other use cases where RCU provides consistent snapshots for free, and quite a few of them may be found in the Linux kernel. However, it is also the case that consistency guarantees are overrated. After all, the finite speed of light sharply limits the degree of consistency that a given data structure can usefully maintain with its corresponding external real-world state. Given that the whole point of many data structures is to track external state, internal consistency all too often becomes nothing more than an expensive fiction. 

**Back to Quick Quiz 3**.

**Quick Quiz 4** : Shouldn't RLU be tried on complex RCU uses such as the Linux kernel's VFS dentry cache-walk code? 

**Answer** : That might well be the case. I therefore have already pointed one of the authors (Matveev) at Neil Brown's excellent LWN series [here](<http://lwn.net/Articles/649115/>), [here](<http://lwn.net/Articles/649729/>), and [here](<http://lwn.net/Articles/650786/>). Matveev's initial response was as follows: "The dcache/dentry + RCU + various locks is really a complex structure... " I take this as a good sign, in that Matveev should not be blinded by overconfidence. 

**Back to Quick Quiz 4**.
