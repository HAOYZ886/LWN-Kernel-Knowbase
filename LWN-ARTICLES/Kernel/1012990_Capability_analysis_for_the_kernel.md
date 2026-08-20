---
title: Capability analysis for the kernel
url: https://lwn.net/Articles/1012990/
date: "March 10, 2025"
category: "Development tools-Static analysis"
author: "By Jonathan Corbet March 10, 2025"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
March 10, 2025

One of the advantages of the Rust type system is its ability to encapsulate requirements about the state of the program in the type system; often, this state includes which locks must be held to be able to carry out specific operations. C lacks the ability to express these requirements, but there would be obvious benefits if that kind of feature could be grafted onto the language. The Clang compiler has made some strides in that direction with its [thread-safety analysis](<https://clang.llvm.org/docs/ThreadSafetyAnalysis.html>) feature; two developers have been independently working to take advantage of that work for the kernel. 

The Clang feature is based on the concept of "capabilities" that a program can be determined — at compile time — to hold (or not) at any given point. Capabilities are typically the address of a data structure; for example, the address of a specific spinlock can be designated as a capability that a program can acquire with a lock operation. Functions can be annotated to indicate that they acquire or release a capability; developers can also indicate that callers of a function must hold (or _not_ hold) a specific capability. 

#### Adding analysis to the kernel

Bart Van Assche posted [a patch series](<https://lwn.net/ml/all/20250206175114.1974171-1-bvanassche@acm.org>) on February 6 showing how Clang's thread-safety feature could be used with the kernel's mutex type. The core of this work can be found in [this patch](<https://lwn.net/ml/all/20250206175114.1974171-7-bvanassche@acm.org>), which annotates the various mutex-related functions; for example, the prototype for `mutex_lock()` and `mutex_unlock()` are modified to be: 

```
void mutex_lock(struct mutex *lock) ACQUIRE(*lock);
        void mutex_unlock(struct mutex *lock) RELEASE(*lock);
```

The first line says that a call to `mutex_lock()` will gain a capability in the form of the pointed-to mutex, while calling `mutex_unlock()` will give up that capability. The `ACQUIRE()` and `RELEASE()` macros are built on top of Clang's lower-level macros; there are [quite a few](<https://lwn.net/ml/all/20250206175114.1974171-4-bvanassche@acm.org>) other macros in the set. With that infrastructure in place, any function that requires a specific mutex to be held can be annotated accordingly; for example: 

```
static struct devfreq_governor *try_then_request_governor(const char *name)
    	REQUIRES(devfreq_list_lock);
```

The compiler will then warn on any call to that function if the possession of the indicated lock (`devfreq_list_lock`) cannot be determined. There is also a series of macros with names like GUARDED_BY() to document that access to specific data (a structure member, for example) requires that a certain mutex be held. Those macros are not actually used in the posted series, though. 

Van Assche's patch set is focused on the `mutex` type, and attempts to annotate usage throughout the entire kernel (though many of the annotations are `NO_THREAD_SAFETY_ANALYSIS`, which disables the analysis because the locking is too complicated for Clang to figure out — a situation that arises frequently). This work culminates in [a massive patch](<https://lwn.net/ml/all/20250206175114.1974171-33-bvanassche@acm.org>) touching over 800 files, which is a significant amount of code churn. The work has already found a number of locking bugs, the fixes for which are included in the series. 

#### An alternative approach

On the same day, Marco Elver posted [a patch set of his own](<https://lwn.net/ml/all/20250206181711.1902989-1-elver@google.com>) with a slightly different approach to using the same feature; that series has since [been updated](<https://lwn.net/ml/all/20250304092417.2873893-1-elver@google.com>), adopting the term "capability analysis" in place of "thread-safety analysis". While Van Assche used a breadth-first approach with mutexes, Elver has gone depth-first with a series that adds annotations for several locking primitives, but which is only active in subsystems that explicitly opt into it. In that way, warnings can be turned on for code where the maintainers and developers are interested in them (and will act on them), while being left off for the rest of the kernel. 

The syntax of the annotations is a little different from Van Assche's approach, but the intent is clearly the same: 

```
void mutex_lock(struct mutex *lock) __acquires(lock);
        void mutex_unlock(struct mutex *lock) __releases(lock);
```

Elver's series, though, goes beyond mutexes to add annotations for spinlocks, [reader-writer locks](<https://lwn.net/Articles/364583/>), [seqlocks](<https://lwn.net/Articles/22818/>), single-bit spinlocks, read-copy-update, [local locks](<https://lwn.net/Articles/828477/>), and [wait/wound mutexes](<https://lwn.net/Articles/548909/>). In many cases, the annotations that already exist for the kernel's [locking correctness validator](<https://docs.kernel.org/locking/lockdep-design.html>) (lockdep) have been reworked to add the needed capability declarations. There is a `__guarded_by()` annotation to document that a lock that must be held to access a specific structure member; its use can be seen in [this patch](<https://lwn.net/ml/all/20250304092417.2873893-27-elver@google.com>) instrumenting the kfence subsystem. The `capability_unsafe()` marker disables checking for a block of code. Most of the new annotations, along with documentation, can be found in [this patch](<https://lwn.net/ml/all/20250304092417.2873893-3-elver@google.com>). 

The other difference found in Elver's approach is the explicit opt-in design, which allows each subsystem to enable or disable the feature. By default, any given subsystem will not be covered by this analysis; that can be changed by adding one or more lines to the subsystem's makefile: 

```
CAPABILITY_ANALYSIS := y
        CAPABILITY_ANALYSIS_foo := y
```

The first line will enable analysis for all code compiled by way of that makefile, while the second will enable it only for the compilation creating `foo.o`. The patch set enables the feature for a number of kernel subsystems, including debugfs, kernel fences, [rhashtables](<https://lwn.net/Articles/751374/>), tty drivers, the [TOMOYO security module](<https://docs.kernel.org/admin-guide/LSM/tomoyo.html>), the crypto subsystem, and more. 

#### What next?

It would appear that the community has a wealth of riches here: two competing patch sets that aim to use the same compiler feature to improve correctness checking within the kernel. Either series can increase confidence that locking is being handled correctly, and both work entirely at compile time, with no run-time overhead. The reception for this work has been quite positive, with the only open question seemingly being which series would be accepted, or whether the kernel might adopt a combination of the two. 

There are no definitive answers to that question, but a clue can be found in the fact that Van Assche has been posting comments (and Reviewed-by tags) for Elver's patch set. Peter Zijlstra has also [tried his hand](<https://lwn.net/ml/all/20250305112041.GA16878@noisy.programming.kicks-ass.net/>) with Elver's work in the scheduler subsystem, saying that ""this is all awesome"". That attempt pointed to some needed changes; it seems Zijlstra also managed to crash the Clang compiler. He later [pointed out](<https://lwn.net/ml/all/20250305161652.GA18280@noisy.programming.kicks-ass.net/>) that the capability analysis works in simple cases, but for ""anything remotely interesting, where you actually want the help, it falls over"". 

Real use in the kernel (and beyond) may well help to drive development work in the Clang community to improve this analysis feature to the point that it can be routinely used to verify locking patterns. Some of that work may need to happen before support for this kind of capability analysis can be added to the kernel. But, since it is an opt-in, compile-time feature, there may well be value to adding it relatively soon, even if it needs further work. The kernel community seems to be hungry for this kind of support.
