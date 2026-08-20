---
title: "Concurrency bugs should fear the big bad data-race detector (part 1)"
url: https://lwn.net/Articles/816850/
date: "April 8, 2020"
category: "Development tools-Linux kernel memory model"
author: "April 8, 2020 (Many contributors)"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

April 8, 2020

(Many contributors)

This article was contributed by Marco Elver, Paul E. McKenney, Dmitry Vyukov, Andrey Konovalov, Alexander Potapenko, Kostya Serebryany, Alan Stern, Andrea Parri, Akira Yokosawa, Peter Zijlstra, Will Deacon, Daniel Lustig, Boqun Feng, Joel Fernandes, Jade Alglave, and Luc Maranget. 

The [first](<https://lwn.net/Articles/793253/>) installment of the "big bad" series described how a compiler can optimize your concurrent program into oblivion, while the [second](<https://lwn.net/Articles/799218/>) installment introduced a tool to analyze small litmus tests for such problems. Those two articles can be especially helpful for training, design discussions, and checking small samples of code. Although such automated training and design tools are welcome, automated code inspection that could locate even one class of concurrency bugs would be even better. In this two-part article, we look at a tool to do that kind of analysis. 

This article focuses on the [Kernel Concurrency Sanitizer](<https://github.com/google/ktsan/wiki/KCSAN>) (KCSAN)—also covered in an earlier [LWN article](<https://lwn.net/Articles/802128/>)—which can locate data races across the entire Linux kernel. This wide scalability does not come for free: KCSAN relies on compiler instrumentation and performs its analysis at runtime, which slows down the kernel considerably. In addition, it can only report data races that actually happen or almost happen during code execution. Nevertheless, KCSAN has already pointed out numerous problems, many of which have now been [fixed](<https://github.com/google/ktsan/wiki/KCSAN#upstream-fixes-of-data-races-found-by-kcsan>). 

Although KCSAN follows the [Linux-Kernel Memory Consistency Model](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/tools/memory-model/Documentation/explanation.txt>) (LKMM), KCSAN can be told to ignore certain classes of data races depending on the preferences of the developers and maintainers, as will be described in part 2\. Such forgiveness is helpful to developers who wish to focus on data races that are not exacerbated by current compilers. Furthermore, KCSAN allows developers to specify certain types of concurrency rules that it also checks for. In this mode KCSAN acts as a thorough concurrency-aware code reviewer, thus providing a much-needed service to the kernel community. 

#### Why care about data races?

The C language evolved independently of concurrency. Consequently, C compilers are permitted to assume that if there is nothing special about a given variable or access, the variable will change only in response to a store by the current thread. Compilers therefore can and do use a variety of optimizations involving load fusing, code reordering, and many others—described in the first installment—that can cause concurrent algorithms to malfunction. 

By [definition](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/tools/memory-model/Documentation/explanation.txt#n1922>), data races occur when there are concurrent conflicting accesses from multiple threads (or tasks or CPUs), at least one of which is a plain (unmarked) C-language access; accesses conflict if they all access the same memory location and at least one performs a write. While KCSAN can enforce that strict definition, by default it treats all aligned writes up to word size, whether marked or not, as atomic, so it is only looking for unmarked reads that race with those writes. 

The wide variety of optimizations used by modern compilers makes it extremely difficult to predict all possible outcomes for all compilers on all architectures. Worse yet, long experience indicates that optimizers will continue becoming increasingly aggressive. In fact, the C11 memory consistency model (described in section 5.1.2.4 of this [specification [PDF]](<http://www.open-std.org/jtc1/sc22/wg14/www/docs/n2478.pdf>)) allows maximal optimizer aggression by stating that data races invoke undefined behavior.

**Quick quiz 1** : Why can't the Linux kernel just use the C11 memory model?   
Answer

For code that does not follow the C11 memory model, the situation is even less clear. In the case of the Linux kernel, the LKMM specifies the expected behavior of concurrent kernel code given production compilers and systems. To this end, LKMM relies on special marked operations (`READ_ONCE()`, `WRITE_ONCE()`, etc.) that tell the compiler which accesses are expected to race, thus preventing the compiler from applying harmful optimizations. For a summary please see the second installment in the series, "Calibrating your fear of big bad optimizing compilers".

**Quick quiz 2** : What's the difference between "data races" and "race conditions"?   
Answer

However, if a data race results in unexpected system behavior then this data race is also a race-condition bug, and it is likely to also be a symptom of a bug in the system's higher-level logic. One common example of such a bug is access to lock-protected shared variables from threads that have failed to acquire the corresponding lock. Developers and maintainers who have appropriately marked their accesses to shared variables will find that KCSAN's reports point to such logic bugs. Furthermore, KCSAN allows developers some control over the warnings issued, which in turn allows those developers to focus KCSAN's automated review on the race conditions of interest. 

#### The Kernel Concurrency Sanitizer

KCSAN is a tool that detects data races as defined by the LKMM, but with control over exactly what sorts of data races are reported. KCSAN is aware of all marked atomic operations that the LKMM defines, as well as operations not yet mentioned by the LKMM, such as [atomic bitmask operations](<https://www.kernel.org/doc/html/v5.5/core-api/atomic_ops.html#atomic-bitmask>). KCSAN also extends the LKMM, for example by providing the `data_race()` marking, which denotes intentional data races and a possible lack of atomicity. 

KCSAN is nothing more or less than a way of carrying out an extremely detailed automated concurrency-aware code inspection. In this way, KCSAN augments the "10,000 eyes" with a set of eyes that do not get tired, and which has been running continuously on a [syzbot](<https://github.com/google/syzkaller/blob/master/docs/syzbot.md>) instance since October 2019\. To get a peek at some of the data races being found, without having to run KCSAN yourself, have a look at this [dashboard](<https://syzkaller.appspot.com/upstream?manager=ci2-upstream-kcsan-gce>). 

KCSAN relies on observing that two accesses happen concurrently. Crucially, there is a desire to (a) increase the chances of observing races (especially for races that manifest rarely), and (b) be able to actually observe them. Those things can be accomplished (a) by injecting various delays, and (b) by using address watchpoints (or breakpoints). 

If memory accesses are deliberately stalled, while a watchpoint is active for that address, then if the watchpoint is observed to fire, two accesses to the same address just raced. This is the approach taken in [DataCollider [PDF]](<http://usenix.org/legacy/events/osdi10/tech/full_papers/Erickson.pdf>) using hardware watchpoints. KCSAN does not use hardware watchpoints, but instead relies on compiler instrumentation and "soft watchpoints". 

In KCSAN, watchpoints are implemented using an efficient encoding that stores access type, size, and address in a long integer; the benefits of using "soft watchpoints" are portability and greater flexibility. KCSAN then relies on the compiler instrumenting plain memory accesses. For each instrumented plain access, KCSAN will: 

  * Check if a matching watchpoint exists; if yes, and at least one access is a write, then a racing access has been encountered.
  * Periodically, if no matching watchpoint exists, set up a watchpoint and stall for a small randomized delay.
  * Also check the data value before the delay, and re-check the data value after delay; if the values mismatch, a race of unknown origin is inferred.

To detect data races where some (but not all) accesses have been marked with an annotation like `READ_ONCE()`, KCSAN also instruments marked accesses, but only to check if a watchpoint exists; i.e. KCSAN never sets up a watchpoint on marked accesses. So if all accesses to a variable that is accessed concurrently are properly marked, KCSAN will never trigger a watchpoint, since it never set one up, and therefore will never report the accesses. 

#### How to use KCSAN

The best use of KCSAN depends on the maintainers, developers, and the code in question. This section covers different classes of code and how KCSAN can best help find potential concurrency bugs, then looks at ways of organizing KCSAN reports. But if you remember only one thing from this section, let it be "Do NOT respond to KCSAN reports by mindlessly adding `READ_ONCE()`, `data_race()`, and `WRITE_ONCE()`." The following sections (and part 2) will give a few reasons why this rule is so important. 

Recent Linux installations provide everything needed to build the kernel with KCSAN, though KCSAN itself has not been merged to the mainline as yet; it is available in linux-next at this point. A compiler upgrade is required for older Linux installations with GCC prior to version 7.3.0 or with Clang prior to version 7.0.0. Given a KCSAN-capable compiler, running your kernel built with `CONFIG_KCSAN=y` might result in the following report: 

```
==================================================================
        BUG: KCSAN: data-race in rcu_torture_reader / run_timer_softirq
    
        read (marked) to 0xffff9ea500543e98 of 8 bytes by task 155 on cpu 4:
         rcu_torture_reader+0x2cb/0x3b0
         kthread+0x1c3/0x1e0
         ret_from_fork+0x35/0x40
    
        write to 0xffff9ea500543e98 of 8 bytes by interrupt on cpu 0:
         run_timer_softirq+0x63c/0x980
         __do_softirq+0xd8/0x2cb
         irq_exit+0xc3/0xd0
         smp_apic_timer_interrupt+0xae/0x230
         apic_timer_interrupt+0xf/0x20
         delay_tsc+0x1b/0x60
         rcu_torture_fwd_prog+0x39d/0xe20
         kthread+0x1c3/0x1e0
         ret_from_fork+0x35/0x40
    
        Reported by Kernel Concurrency Sanitizer on:
        ...
        ==================================================================
```

This shows an eight-byte data race between the functions `rcu_torture_reader()` and `run_timer_softirq()`, with `rcu_torture_reader()` having done a marked read ("`read (marked) to ...`") and `run_timer_softirq()` having done an unmarked write ("`write to ...`"), which by the strict LKMM definition constitutes a data race. 

However, in this case, the data race is not immediately apparent from the source code of these two functions. One approach would be to attempt to work out which data structure resides at address `0xffff9ea500543e98` and another approach is to examine the assembly language at `rcu_torture_reader+0x2cb` and `run_timer_softirq+0x63c`. However, both of these time-honored approaches are labor-intensive and uncertain, not least due to the aggressive inlining and code-motion optimizations practiced by today's compilers. 

A much nicer approach is to build your kernel with `CONFIG_DEBUG_INFO=y`, thus providing debug information that can be used by a variety of tools, for example, `gdb` (alternatives to `gdb` include [`scripts/decode_stacktrace.sh`](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/scripts/decode_stacktrace.sh>) and [`syz-symbolize`](<https://github.com/google/syzkaller/tree/master/tools/syz-symbolize>)): 

```
(gdb) l*rcu_torture_reader+0x2cb
        0xffffffff8114a2bb is in rcu_torture_reader (./include/linux/list.h:784).
        779      * to avoid potential load-tearing.  The READ_ONCE() is paired with the
        780      * various WRITE_ONCE() in hlist helpers that are defined below.
        781      */
        782     static inline int hlist_unhashed_lockless(const struct hlist_node *h)
        783     {
        784             return !READ_ONCE(h->pprev);
        785     }
        786
        787     /**
        788      * hlist_empty - Is the specified hlist_head structure an empty hlist
```

The first of the conflicting accesses is the `READ_ONCE()` in `hlist_unhashed_lockless()`, which was apparently inlined into `rcu_torture_reader()`. This fact might have been difficult to glean from the assembly language. Given this information, the location of the other conflicting access is unsurprising: 

```
(gdb) l*run_timer_softirq+0x63c
        0xffffffff81169aec is in run_timer_softirq (./include/linux/list.h:931).
        926     static inline void hlist_move_list(struct hlist_head *old,
        927                                        struct hlist_head *new)
        928     {
        929             new->first = old->first;
        930             if (new->first)
        931                     new->first->pprev = &new->first;
        932             old->first = NULL;
        933     }
        934
        935     #define hlist_entry(ptr, type, member) container_of(ptr,type,member)
```

The unmarked write on line 931 is the conflicting access, again according to the strict LKMM definition of data race. However, a kernel built with the default KCSAN `Kconfig` options would not report this data race because unmarked writes are considered to be atomic. By default, and at a high level, KCSAN looks for unmarked reads that run concurrently with any sort of write to that same variable. This is less strict than LKMM, which would in addition look for unmarked writes that run concurrently with any sort of read from or write to that same variable. If you want reports according to the strict LKMM definition (as Paul McKenney does when applying KCSAN to RCU), build your kernel with the following additional `Kconfig` options: 

```
CONFIG_KCSAN_ASSUME_PLAIN_WRITES_ATOMIC=n
        CONFIG_KCSAN_REPORT_VALUE_CHANGE_ONLY=n
        CONFIG_KCSAN_INTERRUPT_WATCHER=y
```

These options are discussed in more detail in part 2\. A summary of various options is also available in [`Documentation/dev-tools/kcsan.rst`](<https://git.kernel.org/pub/scm/linux/kernel/git/next/linux-next.git/tree/Documentation/dev-tools/kcsan.rst>). 

#### Summarizing KCSAN reports

It is not unusual for a moderate-length test to produce thousands of KCSAN reports, which does not necessarily generate enthusiasm for working on fixes. However, most of these reports will be duplicates. One way to reduce the number of duplicates is to build your kernel with KCSAN's `KCSAN_REPORT_ONCE_IN_MS` `Kconfig` parameter set to a large value. For example, building with `CONFIG_KCSAN_REPORT_ONCE_IN_MS=10000` will collapse any duplicate reports that occur within a ten-second interval. This can greatly reduce the number of duplicates from a given run; for example, on a 90-minute TREE05 rcutorture scenario, using this `Kconfig` option value reduced the number of KCSAN reports from 6,413 to 3,050. 

However, by default rcutorture runs 16 scenarios, ten of which are SMP, thus potentially producing KCSAN diagnostics. It is sometimes useful to collapse the reports from all of these scenarios and summarize them by function names, for example, using a script like the following: 

```
#!/bin/sh
        grep "BUG: KCSAN: " "$@" | \
                sed -e 's/^\[[^]]*] //' | \
                sort | uniq -c | sort -k1nr
```

Given the console logs ("`$@`" above) collected during an rcutorture run in which each scenario ran for 90 minutes, this script reduced 29,312 KCSAN reports to only 56 lines of output. This is a much more manageable number of reports. Of course, much detail is lost, but this detail can be recaptured by searching the console output for the KCSAN reports of interest. 

By default, KCSAN operates globally, which means that only a small fraction of the reports will normally pertain to the subsystem at hand. In general, it is quite difficult to identify which reports pertain to which subsystem due to inlining and the possibility that a report in a called function is because of the failure of the caller to provide proper synchronization. One of the following approaches might be useful: 

  * Deduplicate as discussed above, and then look at one report from each of the resulting categories. This works well, though it can miss cases where a pair of functions have data races on more than one variable.
  * Deduplicate as discussed above, but look only at reports from categories containing a function defined in the subsystem at hand. This works best when there are a large number of reports even after deduplication, but risks missing important reports due to inlining.
  * Decode the stack traces (for example, using `scripts/decode_stacktrace.sh` or `syz-symbolize`), and look at any report having a stack trace containing a function defined in the subsystem at hand. This is the most thorough approach, but can require looking into a huge number of reports.
  * If only a specific set of known functions defined in the subsystem at hand are of interest, it is possible to tell KCSAN to only report races in those at runtime using its whitelist feature. The next section discusses this in more detail.

#### Interacting with KCSAN at runtime

It is possible to control KCSAN at runtime, which can help to respond to changes in workload or debugging aims by tweaking KCSAN's parameters and by controlling which data races are reported. 

The file `/sys/kernel/debug/kcsan` provides the following interface to KCSAN: 

  * Reading `/sys/kernel/debug/kcsan` returns various runtime statistics, such as the number of data races detected.
  * Writing "on" or "off" to `/sys/kernel/debug/kcsan` allows turning KCSAN on or off, respectively.
  * Writing `!some_func_name` to `/sys/kernel/debug/kcsan` adds `some_func_name` to the report filter list, which (by default) blacklists reporting data races where either one of the top stack frames are a function in the list.
  * Writing either `blacklist` or `whitelist` to `/sys/kernel/debug/kcsan` changes the report filtering behavior. For example, the blacklist feature can be used to silence frequently occurring data races; the whitelist feature can help with reproduction and testing of fixes.

The default configuration parameters are chosen to be conservative, providing overall good performance and race-detection abilities on smaller systems (desktops, workstations, virtual machines). However, large systems, such as servers with more than 64 hardware threads, may require adjustment. 

The core parameters that affect KCSAN's overall performance and bug detection ability are exposed as kernel command-line arguments whose defaults can also be changed via the corresponding `Kconfig` options. All of these arguments and options are related to KCSAN's watchpoint handling. 

  * `kcsan.skip_watch` (`CONFIG_KCSAN_SKIP_WATCH`): Number of per-CPU memory operations to skip before setting up another watchpoint. Setting up watchpoints more frequently will result in the likelihood of races to be observed to increase. This parameter has the most significant impact on overall system performance and race detection ability.
  * `kcsan.udelay_task` (`CONFIG_KCSAN_UDELAY_TASK`): For tasks, the microsecond delay to stall execution after a watchpoint has been set up. Larger values increase the window in which a race may be observed.
  * `kcsan.udelay_interrupt` (`CONFIG_KCSAN_UDELAY_INTERRUPT`): For interrupts, the microsecond delay to stall execution after a watchpoint has been set up. Interrupts have tighter latency requirements, and their delay should generally be smaller than the one chosen for tasks.

On a new system, one may either set the corresponding `Kconfig` option or set them as a boot parameters. For example, on a large system with 64 hardware threads, we would recommend starting with `kcsan.skip_watch=64000`. Then, once the system has booted, the parameter can be tweaked further via `/sys/module/kcsan/parameters/skip_watch`. 

#### Next up

This part has provided an overview of the basic usage of KCSAN and ideas on how to apply it to find data races. In part 2, we will look deeper into KCSAN and how it can be used on various types of code. It will also cover using KCSAN in looking at other types of problems, beyond just those governed by the LKMM. Some strategies, alternative approaches, and known limitations will be covered as well. 

[**Update** : [Part 2](<https://lwn.net/Articles/816854/>) is now available.] 

#### Answers to quick quizzes

**Quick quiz 1** : Why can't the Linux kernel just use the C11 memory model? 

**Answer:** In many cases, the kernel's requirements [cannot easily be cast](<https://lwn.net/Articles/691128/>) into the C11 memory model. While for some parts of the kernel, it could be conceivable to do so, the engineering efforts and resulting inconsistencies make this proposition unattractive today. Note that, the LKMM is still defined at the C-language level, and embedded in the variant of C that the Linux kernel uses today. 

**Back to quick quiz 1**.

**Quick quiz 2** : What's the difference between "data races" and "race conditions"? 

**Answer:** Race conditions occur if concurrently executing operations result in unexpected system behavior. On the other hand, data races are [defined](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/tools/memory-model/Documentation/explanation.txt#n1922>) at the C-language level. Data races can manifest as race-condition bugs either through compiler transformations, or if the high-level logic of the code is buggy to begin with (such as failing to acquire a lock). 

However, not all race conditions are data races, for example if a racing access is incorrectly marked (how KCSAN can also find these is discussed in part 2). For a moment, imagine that we marked every memory access: at this point, there are no more data races and no more KCSAN complaints. However, we may still have race conditions, many of which, such as failing to acquire the necessary locks, can cause the system to misbehave despite the lack of KCSAN complaints. 

Also note that, not all references to "race conditions" imply buggy behavior. Many low-level synchronization mechanisms are meant to resolve race conditions; for example, the reads in race conditions due to unsuccessful sequence-lock reader critical sections will simply be discarded and retried. However, most [definitions](<http://ftp.cs.wisc.edu/pub/paradyn/papers/what-are-races.pdf>) and uses of "race condition" imply buggy behavior; unless otherwise specified, our use of "race condition" follows this notion. 

**Back to quick quiz 2**.

#### Acknowledgments

We would like to thank everyone who has given feedback, comments, or otherwise participated in the work discussed in this article. Some notable discussions and feedback resulted from patches to address data races found by KCSAN: in particular, we would like to thank Eric Dumazet and Qian Cai for addressing numerous data races and their continued feedback, Linus Torvalds, Ingo Molnar, and Herbert Xu for their helpful and critical feedback. We are very grateful to Blake Matheny for their support of this effort.
