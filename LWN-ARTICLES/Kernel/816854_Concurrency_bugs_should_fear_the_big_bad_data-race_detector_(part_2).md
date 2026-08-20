---
title: "Concurrency bugs should fear the big bad data-race detector (part 2)"
url: https://lwn.net/Articles/816854/
date: "April 14, 2020"
category: "Development tools-Linux kernel memory model"
author: "April 14, 2020 (Many contributors)"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

April 14, 2020

(Many contributors)

This article was contributed by Marco Elver, Paul E. McKenney, Dmitry Vyukov, Andrey Konovalov, Alexander Potapenko, Kostya Serebryany, Alan Stern, Andrea Parri, Akira Yokosawa, Peter Zijlstra, Will Deacon, Daniel Lustig, Boqun Feng, Joel Fernandes, Jade Alglave, and Luc Maranget. 

In [part 1](<https://lwn.net/Articles/816850/>) of this article, we gave an overview of the [Kernel Concurrency Sanitizer](<https://github.com/google/ktsan/wiki/KCSAN>) (KCSAN) and looked how it can detect data races in the kernel. KCSAN uses the [definition](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/tools/memory-model/Documentation/explanation.txt#n1922>) of "data race" that is part of the [Linux-Kernel Memory Consistency Model](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/tools/memory-model/Documentation/explanation.txt>) (LKMM), but there is more that KCSAN can do. This concluding part of the article describes other ways that the tool can be used to find data races and other kinds of problems in concurrent code. It provides some ideas on strategies and best practices, briefly considers some alternative approaches, and concludes with some known limitations. 

#### Applying KCSAN to different types of code

When dealing with data races, we need to be aware of the code's requirements and purpose. Some code tolerates data races but other code does not. Where that tolerance also results in improved scalability of the code or design in question, the `data_race()` marking can be applied to any expression where data races are intentional, thus documenting this fact and also telling KCSAN that data races in the expression should be ignored. 

The rest of this section discusses how to get the most out of KCSAN for different types of code. 

Data-racy reads from shared variables that are used only for diagnostic purposes should typically use `data_race()`, since it is normally not a problem if the values are off by a little. Examples include the reads used to construct lockdep reports, monitoring and statistics (including `/proc` and `/sys` output), the argument to `WARN_ON_ONCE()` when the return value is ignored, and other situations where the reads are not in any way an integral part of the core concurrency design for the shared variables in question. There are of course exceptions to this rule. For example, if user-space code requires selected `/sys` output to give a coherent snapshot of in-kernel state, then that output must be considered to be a first-class part of the core concurrency design and must therefore use proper synchronization. 

Reads whose values are checked might also use `data_race()`. Examples include: 

  * Reads that are being fed only into `cmpxchg()` and friends (possibly with some computation on the way) such that `cmpxchg()` is guaranteed to fail if the compiler does anything unexpected with the load. But please keep in mind that a `data_race()` load feeding into a `cmpxchg_relaxed()` might still be subject to [load fusing](<https://lwn.net/Articles/793253/#Load%20Fusing>) on some architectures. Therefore, it is best to capture the return value from the failing `cmpxchg()` for use in the next iteration of the loop, which provides the compiler much less scope for mischievous optimizations. This approach also saves a memory reference in many cases.
  * Reads within a sequence-locking read-side critical section, whose values are ignored unless the subsequent `read_seqretry()` returns false. However, `data_race()` is only needed in sequence-locking read-side critical sections for reads that access variables updated outside of the corresponding write-side critical section. Reads of variables updated only within the write-side critical section are automatically ignored by KCSAN, which understands and automatically recognizes sequence locks.
  * Reads that feed into heuristics, such that occasional errors are compensated for. But again `data_race()` loads are still subject to load fusing, which can result in consistent errors, which in turn are quite capable of breaking heuristics.
  * If compilers are assumed never to [invent stores](<https://lwn.net/Articles/793253/#Invented%20Stores>) just prior to a normal store, then concurrent normal loads might be able to make some assumptions about the value loaded. For example, kernel-space addresses typically have at least a few upper bits set because the low-numbered addresses are normally reserved for user space. In such a system, a comparison to a `NULL` pointer will give the correct answer no matter how the compiler slices and dices the loads and stores.

Reads from and writes to variables that are not supposed to be subject to data races should use plain C-language reads and writes, thus enabling KCSAN to flag bugs involving unintended (and thus likely buggy) data races. Important categories of non-data-racy situations include: 

  * Accesses protected by strict locking.
  * Initialization- and cleanup-time accesses. This covers a wide variety of situations, including the uniprocessor phase of system boot, variables to be used by not-yet-spawned kthreads, structures not yet published to reference-counted or RCU-protected data structures, and the cleanup side of each of these situations.
  * Per-CPU variables that are not supposed to be accessed from other CPUs. This case occurs in RCU, for example.
  * Private per-task variables, including on-stack variables, some fields in the `task_struct` structure, and task-private heap data.

As noted above, data-racy reads for diagnostic accesses to otherwise data-race-free variables should use `data_race()`. The non-diagnostic accesses should **not** be marked. After all, accesses to data-race-free variables need to remain unmarked in order to allow KCSAN to detect buggy concurrent accesses. 

Note that marking accesses racing with accesses marked `data_race()` is optional, but for documentation purposes it is still recommended to do so where appropriate. 

According to the strict LKMM data-race definition, other use cases should use `READ_ONCE()` or stronger for intentionally data-racy reads and `WRITE_ONCE()` or stronger for intentionally data-racy writes. However, there are exceptions depending on taste, and KCSAN can be configured to suit: 

  * Plain reads that race with writes that do not change the value of the shared variable can be forgiven by building with `CONFIG_KCSAN_REPORT_VALUE_CHANGE_ONLY=y` (selected by default). This is useful in cases where a variable is compile-time initialized to (say) zero and then repeatedly set to (say) the value one during execution. In such cases, the compiler would need to be rather demonic to load some other value other than one when running concurrently with a write that left the value unchanged. However, some developers might prefer to check the value so as to avoid same-value stores altogether, thus possibly also improving cache locality.
  * Unmarked writes (aligned and up to word size) can be treated as if they had used `WRITE_ONCE()` by building with `CONFIG_KCSAN_ASSUME_PLAIN_WRITES_ATOMIC=y` (also selected by default). Experience has shown that compilers are much less likely to destructively optimize in-kernel writes than reads. Some developers might therefore choose to use `READ_ONCE()` but omit the corresponding `WRITE_ONCE()`. Other developers might prefer the documentation benefits and long-term peace of mind accruing from explicit use of `WRITE_ONCE()`, [especially those developers whose code stores certain constants](<https://lore.kernel.org/lkml/20190821103200.kpufwtviqhpbuv2n@willie-the-truck/>).

As before, accesses that are intended to be non-data-racy should use plain C-language loads and stores, which allows KCSAN to find accesses that violate your synchronization rules. 

Although KCSAN cannot directly detect unnecessary use of `READ_ONCE()` and `WRITE_ONCE()`, one way of indirectly detecting this is to remove the `READ_ONCE()` or `WRITE_ONCE()` from accesses that are suspected to be unnecessary, and then run KCSAN using a benchmark that has high probability of exercising the suspected race conditions. However, some care is required given that the data raciness of a particular access might depend on `Kconfig` options or kernel boot parameters and `sysfs` settings. 

The list in this section represents best practices at this point in time. Both KCSAN and the best practices surrounding its use can be expected to change as we learn how best to use this data-race detector and also as KCSAN evolves. 

#### Developer/Maintainer data-race strategies

An [earlier article](<https://lwn.net/Articles/793253/#How%20Real%20Is%20All%20This?>) described four approaches to applying markings for shared variables, which can be paraphrased as follows: (1) Never, (2) Clear and present bug, (3) Data races, and (4) Always. The following list describes how to use KCSAN for each option, and adds a fifth option that has resulted from experience using KCSAN. 

  * **Never** : In this case, you may add "`KCSAN_SANITIZE_file.o := n`" to the respective `Makefile` (or "`KCSAN_SANITIZE := n`" for all targets in the `Makefile`). Note that data races between one access having KCSAN enabled, and the other not having it enabled, will result in a "race of unknown origin" report, pointing at the access for which KCSAN was enabled. If you do not want to see any such data races, set `CONFIG_KCSAN_REPORT_RACE_UNKNOWN_ORIGIN=n`.
  * **For any access to any shared variable for which there is a possibility of a data race, and for which it can be clearly shown that specific compiler optimizations could result in bugs** : Analyze most KCSAN reports for vulnerability to compiler optimizations. This is not a one-off task because data races that appear "benign" today, might not [remain benign [PDF]](<https://www.usenix.org/legacy/events/hotpar11/tech/final_files/Boehm.pdf>) in the future. It is up to the developer to prove that the compiler cannot break the code. When carrying out such proofs, whether formal or informal, please recall that plain C-language accesses are subject to load/store fusing, invented loads/stores, dead-code elimination, load/store tearing, and possibly other manipulations from clever compiler developers. If a given data race is deemed to be benign, you can prevent KCSAN from complaining about it via the `data_race()` macro. As long as all conflicting access are marked (either `data_race()` or `READ_ONCE()`, `WRITE_ONCE()`, etc.), KCSAN will be silent. Alternatively, tell KCSAN to ignore all accesses in a function via the `__no_kcsan` function attribute.
  * **For any access to a shared variable for which there is a possibility of a data race for at least one of those accesses** : Enable the most aggressive KCSAN reporting and act on all KCSAN reports. All accesses to any shared variable involved in a KCSAN data race report needs to use any one of the appropriate marked atomic operations (`READ_ONCE()`, `WRITE_ONCE()`, etc., but avoid `data_race()` except in diagnostic code).
  * **For all accesses to all shared variables** : KCSAN is a bit more forgiving than this, in that it only reports true concurrent access. However, most of the time, if KCSAN reports a data race on a shared variable, you can then apply the appropriate marking to all accesses to that variable, even if they may not actually race with another access.
  * **For any access to any shared variable for which there is a possibility of a data race, except for certain cases where a reasonable compiler (as opposed to a theoretical demonic compiler that just barely adheres to the standard) would not break concurrent algorithms** : Act on most KCSAN reports, but also configure KCSAN to match your notion of what constitutes a reasonable compiler (but note that the defaults are quite forgiving of data races). If some future compiler becomes unreasonable and thus applies optimizations that break concurrent code, the Linux community would disable those optimizations by specifying compiler flags. However, it is worthwhile to look at other accesses to the variables in KCSAN reports because, similar to lockdep, KCSAN reports only those data races exercised by actual executions.

Patch submissions based on KCSAN reports also need a strategy: As noted in part 1, you should not respond to KCSAN reports by mindlessly adding `READ_ONCE()`, `data_race()`, and `WRITE_ONCE()`. Instead, a patch addressing a KCSAN report must clearly identify the fix's approach and why that approach is appropriate. 

Of course, the KCSAN report should be included in the commit log, preferably with file/line numbers; simplify the report, where appropriate, by removing irrelevant portions of the stack traces, for example. Furthermore, the commit log should include a detailed description of the problem that the KCSAN report identified and the reasoning behind the fix. Where possible, this reasoning can reference or excerpt the comments and documentation defining the design rules violated by the old code. In a few cases, this detailed description might obviate the KCSAN report, but in that case, please do at least credit KCSAN. 

If the commit's effect is to merely silence the warning with no other algorithmic change, many maintainers will treat the commit with great suspicion—and rightly so. Therefore, such commits should include a clear explanation why silencing the warning is the right thing to do. 

Finally, the commit log should include instructions to reproduce the data race, if possible, along with any non-default `Kconfig` options. 

#### Taking KCSAN beyond LKMM

Data races are often symptoms of logic bugs, which is why KCSAN's ability to locate them is so valuable. Therefore, as noted earlier, reacting to each and every KCSAN report by mindlessly adding a `READ_ONCE()` or a `WRITE_ONCE()` is a grave mistake, especially when the code being modified has multiple callers. The value of these reports is instead that they inform the developer of unexpected concurrent accesses, and the correct response might instead be to apply proper synchronization, for example, by acquiring the corresponding lock. In this case, adding the `READ_ONCE()` would resolve the data race, but would simply obscure the underlying bug without actually helping anything at all. So again please think carefully about KCSAN's reports instead of mindlessly reacting to them. 

This is why the previous section recommends leaving accesses unmarked in many cases: leaving them unmarked means that a race-condition bug will result in a data race, permitting KCSAN to find it for you. 

Although KCSAN's ability to identify data races is extremely useful, data-race detection is just the tip of its iceberg. For example, suppose that there is a bug in which a pointer is leaked from an RCU read-side critical section, perhaps as follows: 

```
// Reader:
    	p = NULL;
    	rcu_read_lock();
    	list_for_each_entry_rcu(p, list1_head, list1) {
    		if (p->key = mykey)
    			break;
    	}
    	if (p)
    		do_something_with(p);
    	rcu_read_unlock(); // At this point, *p is no longer protected.
    	do_something();
    	if (p)
    		do_something_else_with(p); //BUG:  Leaked pointer!
```

Suppose further that the updater splices `list1_head` onto `list2_head` using `list_splice_init_rcu()` as follows: 

```
// Updater:
    	list_del_rcu(&fp->list1);
    	list_splice_init_rcu(list1_head, list2_head, synchronize_rcu);
```

**Quick quiz 3** : But isn't it rather unlikely that `do_something()` will consume enough time for the grace period to complete?   
Answer

By the time that `list_splice_init_rcu()` returns, RCU readers might be legitimately traversing all of the elements on `list2_head`, including those recently on `list1_head`, thus preventing KCSAN from providing any useful information. However, there is a point within `list_splice_init_rcu()` where non-leaky RCU readers will not have access to the elements from `list1_head`. At this point, calls to `ASSERT_EXCLUSIVE_ACCESS()`, as shown below, will allow KCSAN to detect leaky RCU readers: 

```
static inline void __list_splice_init_rcu(struct list_head *list,
    						  struct list_head *prev,
    						  struct list_head *next,
    						  void (*sync)(void))
    	{
    		struct list_head *first = list->next;
    		struct list_head *last = list->prev;
    
    		INIT_LIST_HEAD_RCU(list);
    		sync();
    		ASSERT_EXCLUSIVE_ACCESS(*first); // KCSAN, any off-CPU accesses?
    		ASSERT_EXCLUSIVE_ACCESS(*last);
    		last->next = next;
    		rcu_assign_pointer(list_next_rcu(prev), first);
    		first->prev = prev;
    		next->prev = last;
    	}
    
    	static inline void list_splice_init_rcu(struct list_head *list,
    						struct list_head *head,
    						void (*sync)(void))
    	{
    		if (!list_empty(list))
    			__list_splice_init_rcu(list, head, head->next, sync);
    	}
```

KCSAN can also help when memory is freed into an emergency reserve that is used only under out-of-memory conditions. For example, consider this example: 

```
if (atomic_dec_and_test(&p->refcount)) {
    		ASSERT_EXCLUSIVE_ACCESS(*p);  // KCSAN, any off-CPU accesses?
    		do_some_cleanup(p);
    		list_add(&p->freelist, &emergency_reserve);
    	}
```

**Quick quiz 4** : Why not also apply KCSAN in the more common case where the list elements are instead freed via `kfree()`? After all, KCSAN should be able to directly detect the resulting use-after-free bug, shouldn't it?   
Answer

The call to `ASSERT_EXCLUSIVE_ACCESS()` allows KCSAN to detect buggy concurrent accesses that failed to acquire the needed reference. 

There is another set of patterns, where concurrent reads are allowed, but concurrent writes are forbidden; bugs in the implementation of these patterns man not be data races. Consider the case where we are only supposed to have a single writer, but multiple concurrent readers; to avoid data races, all these accesses must be marked. Concurrent _marked_ writes, however, that are racing with the single writer are bugs. Unfortunately, due to being marked, they are no longer data races. Could KCSAN help us in this case? 

For example, a common pattern involves a variable that is updated only when its lock is held, but that can be read locklessly. Given that there are lockless marked reads scattered hither and yon, it is all too easy to add a buggy lockless _marked_ write, and such writes can be quite difficult to find, especially in cases involving locks held across deeply nested function calls. Of course, the `lockdep_is_held()` function can be extremely useful in these situations, but in the all-too-common case where the variable in question is a non-uniquely named field in a structure, it can be surprisingly hard to place calls to this function everywhere it is needed and nowhere that it is not. 

Again, KCSAN can help: 

```
spin_lock(&my_lock);
    	WRITE_ONCE(read_me_locklessly, foo);
    	ASSERT_EXCLUSIVE_WRITER(read_me_locklessly); // KCSAN, lockless writes?
    	spin_unlock(&my_lock);
```

**Quick quiz 5** : So KCSAN can unconditionally locate bugs stemming from unauthorized lockless writes?   
Answer

Of course, `ASSERT_EXCLUSIVE_WRITER()` can also be used to detect off-CPU writes to per-CPU variables that are supposed to be read-only to other CPUs, as well as similar bugs that do not directly result in data races. 

We are confident that continued experience with KCSAN will result in additional tricks for finding other types of concurrency bugs. 

In addition, we expect that KCSAN will continue to gain more capabilities. One late-breaking example is a bit-granular variant of `ASSERT_EXCLUSIVE_WRITER()` called `ASSERT_EXCLUSIVE_BITS()`. This supports bitmask use cases where some bits are read-only or are supposed to be modified only under the protection of an exclusive lock, while others can be modified concurrently. Note again that a knee-jerk application of `READ_ONCE()` to silence KCSAN would be obscuring a real bug, which is most definitely not what we want. 

#### Alternative approaches

An alternative data race detection approach for the kernel can be found in the [Kernel Thread Sanitizer (KTSAN)](<https://github.com/google/ktsan/wiki>). KTSAN is a data-race detector that explicitly establishes which memory operations are ordered (this is also known as a "[happens-before [PDF]](<http://lamport.azurewebsites.net/pubs/time-clocks.pdf>)" [data-race detector [PDF]](<http://static.googleusercontent.com/media/research.google.com/en//pubs/archive/35604.pdf>)). This information can then be used to determine data races as [defined by the LKMM](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/tools/memory-model/Documentation/explanation.txt#n1922>). 

To build a correct happens-before relation, KTSAN must be aware of all ordering rules of the LKMM and the synchronization primitives. Unfortunately, any omission leads to large numbers of false positives; this is especially detrimental in the context of the kernel, which includes numerous custom synchronization mechanisms. To track the happens-before relation, KTSAN's implementation requires metadata for each memory location (shadow memory), which corresponds to four extra pages of shadow memory for each page of kernel memory; that can translate into overhead of tens of GiB on a large system. In contrast, KCSAN's memory overhead is only a few MiB depending on configuration. Relying on the happens-before relation, however, allows KTSAN to detect more subtle bugs, such as missing memory barriers (implicit or otherwise). KCSAN is currently oblivious to any memory-ordering guarantees and simply assumes that memory barriers are placed correctly (and developers should therefore carefully consider the required memory-ordering requirements that remain unchecked). 

Another approach relies on [lockdep](<https://www.kernel.org/doc/Documentation/locking/lockdep-design.txt>), which can sometimes point to data races due to failure to acquire the necessary locks. This, however, requires explicit lockdep annotations. KCSAN can detect the resulting data races without any added annotation, but on the other hand, for KCSAN to diagnose the problem, both ends of the data race must execute at about the same time. Therefore, lockdep can detect more bugs with less test time if the required annotations are in place, but KCSAN can detect those bugs without the annotations, albeit requiring more test time. These approaches should thus be used in concert, with lockdep annotations being added to code that KCSAN has shown is prone to being invoked without holding the required locks. 

**Quick quiz 6** : Is there a place for a user-space CSAN?   
Answer

All of these tools use dynamic techniques, which means that none of them will diagnose problems on code paths that are not reached by the test cases. On the other hand, experience to date indicates that dynamic analysis approaches (especially when paired with a coverage-guided fuzzer such as [syzkaller](<https://github.com/google/syzkaller>)) are more effective than static-analysis approaches at locating these classes of bugs in large code bases. For example, the LKMM tooling can more deterministically detect data races and other concurrency issues, but is limited to only a few tens of lines of code, six orders of magnitude short of what is required for the Linux kernel. Nevertheless, the LKMM is extremely useful at design time and for educational purposes. 

#### Summary and known limitations

Clearly, bugs involving data races should fear a data-race detector, but KCSAN's `ASSERT_EXCLUSIVE_WRITER()` and `ASSERT_EXCLUSIVE_ACCESS()` macros provide artificial data races that can aid in the detection of other forms of race conditions. KCSAN-detectable concurrency bugs discussed earlier in this article include: 

  * Failing to acquire needed locks.
  * Off-CPU accesses to CPU-private per-CPU variables.
  * Use-after-free bugs for non-heap memory.
  * Leaking pointers from RCU read-side critical sections.

These types of bugs can be quite challenging to find by testing or by manual code review. Therefore, KCSAN's automated code review should prove to be helpful. 

Of course, KCSAN does have limitations: 

  * As a runtime tool, KCSAN detects races only between accesses that execute on different CPUs at about the same time. (But note that a recent KCSAN patch allows checking for data races between accesses at process level and in interrupt handlers within the same CPU.)
  * Although KCSAN provides a number of configuration options, it is currently unable to analyze the compiled code's use of a loaded value. We expect KCSAN to become more capable over time, but features requiring changes to compilers will take some time to reach KCSAN's users.
  * Manual analysis is sometimes required to work out which subsystem caused a given KCSAN report. This is, of course, a limitation shared by all tools that display stack traces.
  * KCSAN does not understand much about your synchronization design, so deep knowledge of the code is required to respond meaningfully to KCSAN reports.

Despite these limitations, KCSAN has proven to be useful, locating a number of subtle and hard-to-spot concurrency bugs in the short time it has been available. We expect that its tireless and thorough automated reviews of concurrent code will be a valuable addition to the Linux kernel's storied 10,000 code-review eyes. 

#### Answers to quick quizzes

**Quick quiz 3** : But isn't it rather unlikely that `do_something()` will consume enough time for the grace period to complete? 

**Answer:** True enough. After all, KCSAN cannot help unless the data-racy accesses happen at about the same time. However, there are ways to help KCSAN help you: 

  * Test on a `CONFIG_PREEMPT=y` kernel, so that the end of an un-nested `rcu_read_unlock()` implies a quiescent state.
  * Add test code to `rcu_read_unlock()` that infrequently adds random delays when not within another RCU read-side critical section.
  * For testing, pass `synchronize_rcu_expedited()` to `list_splice_init_rcu()` instead of `synchronize_rcu()`.

**Back to quick quiz 3**.

**Quick quiz 4** : Why not also apply KCSAN in the more common case where the list elements are instead freed via `kfree()`? After all, KCSAN should be able to directly detect the resulting use-after-free bug, shouldn't it? 

**Answer:** Yes, it might, but [KASAN](<https://www.kernel.org/doc/html/latest/dev-tools/kasan.html>) is normally a better choice than KCSAN for use-after-free bugs. However, finding racy use-after-free bugs with KASAN can be quite challenging, in part because KASAN has no way to know that the bug was in fact due to a race condition rather than a deterministic use-after-free situation. In addition, KASAN cannot help when the memory is not free, but instead placed into a private pool as in the earlier example. However, KCSAN can still help if you invoke `ASSERT_EXCLUSIVE_ACCESS()` just after removing others' access to the memory. 

In short, we recommend starting with KASAN when tracking use-after-free bugs, but looking to KCSAN in those rare cases where KASAN is flummoxed due to race conditions or private memory pools. 

**Back to quick quiz 4**.

**Quick quiz 5** : So KCSAN can unconditionally locate bugs stemming from unauthorized lockless writes? 

**Answer:** We are sorry, but, although we believe that KCSAN will prove to be an extremely valuable tool, it is not magic. 

KCSAN is a runtime tool, which means that it will not detect bugs in code paths that are not actually executed. Furthermore, the unauthorized-write bug must execute reasonably close to the time that the `ASSERT_EXCLUSIVE_WRITER()` is executed, and that write must occur on some other CPU. Additionally, KCSAN does impose significant overhead, and that overhead might well turn your bug into a heisenbug. 

Again, KCSAN is not magic. If it is magic that you are looking for, we refer you to any number of works of fiction, with "Harry Potter", "A Wizard of Earthsea", and the television series "Grimm" coming immediately to mind. And sometimes escaping into fiction for awhile is a good way to free your mind (oops, no, that is "The Matrix") to locate the bug. But in our experience, a good night's sleep usually works even better. 

**Back to quick quiz 5**.

**Quick quiz 6** : Is there a place for a user-space CSAN? 

**Answer:** User space has different constraints than the kernel. For user-space C and C++ programs, we can generally rely on a limited set of synchronization primitives (pthreads) and adherence to the standard memory models (C11, C++11). As such, [ThreadSanitizer (TSAN)](<https://github.com/google/sanitizers/wiki/ThreadSanitizerCppManual>) is a more complete solution in user space that is already well-established. A user-space variant of CSAN would therefore not add much value over TSAN. Note that, while KCSAN as implemented in the Linux kernel provides additional facilities to express properties where bugs won't manifest as data races (`ASSERT_EXCLUSIVE` macros), fundamentally TSAN could provide similar facilities for user space. In other contexts, where TSAN is not yet implemented, and constraints are closer to the Linux kernel, such as firmware, or specialized OSes, CSAN would be an attractive alternative. 

**Back to quick quiz 6**.

#### Acknowledgments

We would like to thank everyone who has given feedback, comments, or otherwise participated in the work discussed in this article. Some notable discussions and feedback resulted from patches to address data races found by KCSAN: in particular, we would like to thank Eric Dumazet and Qian Cai for addressing numerous data races and their continued feedback, Linus Torvalds, Ingo Molnar, and Herbert Xu for their helpful and critical feedback. We are very grateful to Blake Matheny for their support of this effort.
