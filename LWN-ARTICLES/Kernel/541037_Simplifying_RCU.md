---
title: Simplifying RCU
url: https://lwn.net/Articles/541037/
date: "March 6, 2013"
category: "Read-copy-update"
author: "March 6, 2013 This article was contributed by Paul McKenney"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

March 6, 2013

This article was contributed by Paul McKenney

Read-copy update (RCU) is a synchronization mechanism in the Linux kernel that allows extremely efficient and scalable handling of read-mostly data. Although RCU is quite effective where it applies, there have been some [concerns about its complexity.](<https://lwn.net/Articles/522262/>) One way to simplify something is to eliminate part of it, which is what is being proposed for RCU. 

One source of RCU's complexity is that the kernel contains no fewer than four RCU implementations, not counting the three other special-purpose RCU flavors (sleepable RCU (SRCU), RCU-bh, and RCU-sched, which are [covered here](<https://lwn.net/Articles/264090/#RCU%20has%20a%20Family%20of%20Wait-to-Finish%20APIs>)). The four vanilla implementations are selected by the `SMP` and `PREEMPT` kernel configuration parameters: 

  1. `!SMP && !PREEMPT`: `TINY_RCU`, which is used for embedded systems with tiny memories (tens of megabytes). 
  2. `!SMP && PREEMPT`: `TINY_PREEMPT_RCU`, for deep sub-millisecond realtime response on small-memory systems. 
  3. `SMP && !PREEMPT`: `TREE_RCU`, which is used for high performance and scalability on server-class systems where scheduling latencies in milliseconds are acceptable. 
  4. `SMP && PREEMPT`: `TREE_PREEMPT_RCU`, which is used for systems requiring high performance, scalability, _and_ deep sub-millisecond response. 

**Quick Quiz 1** : Since when is ten megabytes of memory _small_???   
Answer

The purpose of these four implementations is to cover Linux's wide range of hardware configurations and workloads. However, although `TINY_RCU`, `TREE_RCU`, and `TREE_PREEMPT_RCU` are heavily used for their respective use cases, `TINY_PREEMPT_RCU`'s memory footprint is not all that much smaller than that of `TREE_PREEMPT_RCU`, especially when you consider that `PREEMPT` itself expands the kernel's memory footprint. All of those `preempt_disable()` and `preempt_enable()` invocations now generate real code. 

The size for `TREE_PREEMPT_RCU` compiled for x86_64 is as follows: 

```
text    data     bss     dec     hex filename
       1541     385       0    1926     786 /tmp/b/kernel/rcupdate.o
      18060    2787      24   20871    5187 /tmp/b/kernel/rcutree.o
```

That for `TINY_PREEMPT_RCU` is as follows: 

```
text    data     bss     dec     hex filename
       1205     337       0    1542     606 /tmp/b/kernel/rcupdate.o
       3499     212       8    3719     e87 /tmp/b/kernel/rcutiny.o
```

If you really have limited memory, you will instead want `TINY_RCU`: 

```
text    data     bss     dec     hex filename
        963     337       0    1300     514 /tmp/b/kernel/rcupdate.o
       1869      90       0    1959     7a7 /tmp/b/kernel/rcutiny.o
```

This points to the possibility of dispensing with `TINY_PREEMPT_RCU` because the difference in size is not enough to justify its existence. 

**Quick Quiz 2** : Hey!!! I use `TINY_PREEMPT_RCU`! What about me???   
Answer

Of course, this needs to be done in a safe and sane way. Until someone comes up with that, I am taking the following approach: 

  1. Poll LKML for objections ([done](<https://lkml.org/lkml/2012/11/12/545>): the smallest `TINY_PREEMPT_RCU` system had 128 megabytes of memory, which is enough that the difference between `TREE_PREEMPT_RCU` and `TINY_PREEMPT_RCU` is 0.01% of memory, namely, down in the noise). 
  2. Update RCU's Kconfig to once again allow `TREE_PREEMPT_RCU` to be built on `!SMP` systems (available in 3.9-rc1 or by applying this [patch](<https://lkml.org/lkml/2013/1/27/1>) for older versions). 
  3. Alert LWN's readers to this change (you are reading it!). 
  4. Allow time for testing and for addressing any issues that might be uncovered. 
  5. If no critical problems are uncovered, remove `TINY_PREEMPT_RCU`, which is currently planned for 3.11. 

Note that the current state of Linus's tree once again allows a choice of RCU implementation in the `!SMP && PREEMPT` case: either `TINY_PREEMPT_RCU` or `TREE_PREEMPT_RCU`. This is a transitional state whose purpose is to allow an easy workaround should there be a bug in `TREE_PREEMPT_RCU` on uniprocessor systems. From 3.11 forward, the choice of RCU implementation will be forced by the values selected for `SMP` and `PREEMPT`, once again adhering to the dictum of No Unnecessary Knobs. 

If all goes well, this change will remove about 1,000 lines of code from the Linux kernel, which is a worthwhile reduction in complexity. So, if you currently use `TINY_PREEMPT_RCU`, please go forth and test `TREE_PREEMPT_RCU` on your hardware and workloads. 

#### Acknowledgments

I owe thanks to Josh Triplett for suggesting this approach, and to Jon Corbet and Linus Torvalds for further motivating it. I am grateful to Jim Wasko for his support of this effort. 

####  Answers to Quick Quizzes

**Quick Quiz 1** : Since when is ten megabytes of memory _small_??? 

**Answer** : As near as I can remember, [Rip](<http://en.wikipedia.org/wiki/Rip_Van_Winkle>), since some time in the early 1990s. 

**Back to Quick Quiz 1**.

**Quick Quiz 2** : Hey!!! I use `TINY_PREEMPT_RCU`! What about me??? 

**Answer** : Please download Linus's current git tree (or 3.9-rc1 or later) and test `TREE_PREEMPT_RCU`, reporting any problems you encounter. Alternatively, try disabling `PREEMPT`, thus switching to `TINY_RCU` for an even smaller memory footprint, relying on improvements in the non-realtime kernel's latencies. Either way, silence will be interpreted as assent! 

**Back to Quick Quiz 2**.
