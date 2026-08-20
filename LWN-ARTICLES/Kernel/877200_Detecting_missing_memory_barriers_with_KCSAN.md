---
title: Detecting missing memory barriers with KCSAN
url: https://lwn.net/Articles/877200/
date: "December 2, 2021"
category: "Development tools-Sanitizers; Lockless algorithms"
author: "By Jonathan Corbet December 2, 2021"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
December 2, 2021

Writing (correct) concurrent code that uses locking to avoid race conditions is difficult enough. When the objective is to use [lockless algorithms](<https://lwn.net/Articles/844224/>), relying on memory barriers instead of locks to eliminate locking overhead, the problem becomes harder still. Bugs are easy to create and hard to find in this type of code. There may be some help on the way, though, in the form of [this patch set](<https://lwn.net/ml/linux-kernel/20211130114433.2580590-1-elver@google.com/>) from Marco Elver that enhances the [Kernel Concurrency Sanitizer](<https://www.kernel.org/doc/html/latest/dev-tools/kcsan.html>) (KCSAN) with the ability to detect some types of missing memory barriers. 

KCSAN works in a statistical manner by watching accesses to specific memory addresses and trying to detect racy patterns; the algorithm used is described in [this article](<https://lwn.net/Articles/802128/>). In its current form, though, KCSAN can only catch certain types of race conditions, specifically those that arise from locking errors. Other types of races remain invisible to this tool, including a number that can arise in incorrect lockless code. KCSAN is, by design, blind to the kinds of problems that occur when CPUs and memory controllers reorder the visibility of memory writes. 

Consider this code example, taken from [this documentation patch](<https://lwn.net/ml/linux-kernel/20211130114433.2580590-10-elver@google.com/>) in Elver's series (and lightly edited): 

```
int x, flag;
        void T1(void)
        {
            x = 1;                  // data race!
            WRITE_ONCE(flag, 1);    // should be: smp_store_release(&flag, 1)
        }
    
        void T2(void)
        {
            while (!READ_ONCE(flag))    // should be: smp_load_acquire(&flag)
    	    ;
            ... = x;                    // data race!
        }
```

At a first glance, this code appears correct. `T1()` stores a value into the variable `x` then sets `flag` to indicate that `x` is valid. The other thread, running in `T2()`, does not attempt to read `x` until `flag` is set, so it should always proceed with a valid value. There is only one little problem: the lack of memory barriers gives the CPU the permission to reorder those operations, which appear to be independent; the write to `x` in `T1()` could, in fact, be visible to the rest of the system _after_ the write to `flag`. That could cause `T2()` to proceed thinking it has a valid value of `x` when the real value is not yet visible to that thread. 

Correct code would have used `smp_store_release()` to write `flag`; that would ensure that all writes done prior to that store would be visible globally before the store to `flag` becomes visible. Similarly, `smp_load_acquire()` is needed to read `flag` in a way that doesn't allow later reads to be reordered to happen earlier. Barriers almost always need to be paired in this way to work properly; omitting either half of the pair creates erroneous code. 

This kind of bug can be difficult for developers implementing lockless algorithms to avoid; the need for a memory barrier for a specific access is not always obvious. Code containing such bugs can work just fine with all of the developer's tests, only to fail on a handful of production systems in obscure settings. That is why some developers, once they understand the challenges of lockless programming, conclude that dedicating their career to implementing annoying popups in JavaScript isn't such a bad thing after all. 

KCSAN, in current kernels, cannot detect this race either. A system running under KCSAN may decide that the store to `x` in `T1()` is interesting and worthy of watching, but the way that KCSAN watches work will prevent the race from occurring. KCSAN will start monitoring accesses to `x` and, while that watch exists, it will delay the further execution of `T1()` to see if any racy accesses happen. `T1()` will only proceed (and set `flag`) once the watch concludes. KCSAN will thus delay the write to `flag`, causing `T2()` to wait, until the watch concludes. So the racy access cannot happen as long as KCSAN is watching, and will go undetected. 

The new code makes one seemingly simple change to try to detect this kind of problem — though that apparent simplicity is belied by the fact that a 25-part patch series is required to implement it. Rather than just forgetting about `x` after the watch period ends, KCSAN will repeat the watch after every subsequent memory access until either a memory barrier is encountered or the function returns. In the case described above, KCSAN will watch `x` again _after_ the assignment to `flag`, essentially simulating the reordering of the writes to those two variables. In essence, this repeated watch is seeing what will happen if the write to `x` becomes visible later than the developer expects. Any access to `x` seen on repeated watches is still racy, since no memory barrier has been executed to ensure correct ordering. So KCSAN will now detect the racy read of `x` in `T2()` and raise the alarm. 

This algorithm can detect a set of race conditions caused by missing barriers, but not all of them. Most notably, it can test the effects of delaying an access to shared data — such as delaying the visibility of the write to `x` in the example above — but not the effects of executing that access earlier than the developer expects. But, even if its coverage is incomplete, the improved KCSAN may well be able to prevent a number of barrier-related bugs from getting to the point where they impact users. That may make lockless algorithms a bit more accessible to non-superhuman developers and might even free a few of them from the specter of a future devoted to JavaScript.
