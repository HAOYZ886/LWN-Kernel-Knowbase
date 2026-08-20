---
title: Restartable sequences in glibc
url: https://lwn.net/Articles/883104/
date: "January 31, 2022"
category: Restartable sequences
author: "By Jonathan Corbet January 31, 2022"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
January 31, 2022

"Restartable sequences" are small segments of user-space code designed to access per-CPU data structures without the need for heavyweight locking. It is a relatively obscure feature, despite having been supported by the Linux kernel since the 4.18 release. Among other things, there is no support in the GNU C Library (glibc) for this feature. That is about to change with the upcoming [glibc 2.35](<https://sourceware.org/glibc/wiki/Release/2.35>) release, though, so a look at the user-space API for this feature is warranted. 

The kernel makes extensive use of per-CPU data structures to avoid locking. This technique works well if the kernel takes care to disable preemption while those data structures are being manipulated; as long as a task running in the kernel has exclusive access to the data, it can safely make changes. It would be nice to be able to use similar techniques in user space, but user-space code lacks the luxury of being able to disable preemption. So a different approach, which relies on detecting rather than preventing preemption, must be used. 

#### A restartable-sequences refresher

That approach is restartable sequences, which were first [proposed](<https://lwn.net/Articles/650333/>) by Paul Turner in 2015, then later [pursued](<https://lwn.net/Articles/697979/>) by Mathieu Desnoyers and [merged](<http://git.kernel.org/linus/d7822b1e24f2>) in 2018. Restartable sequences rely on a couple of simple rules for the creation of safe, lock-free critical sections. The first rule is that the critical section cannot make any changes to the protected data structure that are visible to other threads until the final instruction in that section. That last instruction will typically be a pointer assignment making the new state of things visible. The other rule is that the section can be interrupted at any time prior to that last instruction; when that happens, the code must be able recover and restart the operation from the beginning. 

Using restartable sequences is a bit tricky because user space must be able to tell the kernel when such a sequence is running. Executing a system call would defeat the purpose of the entire exercise, though; at that point, the thread might as well just grab a lock. So, instead, restartable sequences are managed with a special region of memory shared between user space and the kernel. Specifically, user space sets up a special structure, [`struct rseq`](<https://elixir.bootlin.com/linux/v5.16.3/source/include/uapi/linux/rseq.h#L56>), and informs the kernel of this structure using the `rseq()` system call. The structure is a bit complex, but at its core is field called `rseq_cs`, which is a pointer to [a structure also called `rseq_cs`](<https://elixir.bootlin.com/linux/v5.16.3/source/include/uapi/linux/rseq.h#L40>), containing the description of a critical section: 

```
struct rseq_cs {
    	__u32 version;
    	__u32 flags;
    	__u64 start_ip;
    	__u64 post_commit_offset;
    	__u64 abort_ip;
        };
```

To set up a critical section, a user-space thread fills in an `rseq_cs` structure, setting `start_ip` to the address of the first instruction in that section. The `post_commit_offset` is the length of the critical section in bytes; when added to `start_ip` the result is the first instruction after the end of the section. `abort_ip`, instead, is the address of the instruction to jump to if the sequence is interrupted (via preemption or CPU migration, for example) before it completes. `version` should be zero, and the `flags` field can be used to tweak restart behavior; some information on that can be found in [this man page source](<https://git.kernel.org/pub/scm/libs/librseq/librseq.git/tree/doc/man/rseq.2>). 

Actually running the critical section is a matter of storing the address of the `rseq_cs` structure into the `rseq` structure that was registered with the kernel; this should be done just prior to entry into the section. Whenever the kernel preempts the thread, it will check the instruction pointer to see whether the critical section was executing at the time; if so, when the thread resumes execution, it will jump to the `abort_ip` address, at which point it should recover and try again. 

One potential problem with the restartable-sequences ABI is that any given thread can only register a single `rseq` structure with the kernel. Even checking a single structure adds a bit of overhead to the hottest parts of the scheduler; checking a list of them would be unacceptable. The restriction makes sense, but it does pose a problem in situations where there might be more than one user of restartable sequences in a thread; some of them might be buried inside libraries and invisible to users of those libraries, perhaps several layers up the call stack. For restartable sequences to be a reliable mechanism, there must be a way to prevent these users from stepping on each other's toes. 

#### The GNU C Library's approach

If glibc is to expose restartable sequences to its users, it must have a plausible answer to the sharing problem. The [implementation](<https://lwn.net/ml/libc-alpha/cover.1638880888.git.fweimer@redhat.com/>) put together by Florian Weimer takes the approach of putting glibc in the middle for users of this mechanism. Thus, the registration of the `rseq` structure with the `rseq()` system call is done by glibc itself during initialization; by the time user code runs, that setup will have already been performed. Should an application want to perform its own registration (and not use the glibc support at all), the `glibc.pthread.rseq` [tunable](<https://www.gnu.org/software/libc/manual/html_node/Tunables.html>) can be used to disable the automatic registration. 

Applications using restartable sequences via glibc should include `<sys/rseq.h>`. This header defines the `rseq` and `rseq_cs` structures and a few important variables, the first of which is `__rseq_size`. That will be the size of the `rseq` structure registered by the library, or zero if registration didn't happen for whatever reason (no support in the kernel or disabled, for example). 

Finding the `rseq` structure registered by glibc is not quite as straightforward as one might think. It is stored in the thread control block (TCB) maintained by the library; specifically, it can be found at an offset of `__rseq_offset` bytes from the thread pointer. Actually getting at the thread pointer is an architecture-specific affair, though; GCC offers `__builtin_thread_pointer()` for some architectures but not all. As it happens, x86 is one of the exceptions; there the thread pointer is stored in the FS register and applications must fetch it themselves. 

The glibc-registered `rseq` structure is shared by all users within a given thread, but each user should create its own `rseq_cs` structure describing its critical section. Immediately prior to entering its critical section, a thread should store the address of its `rseq_cs` structure into the `rseq_cs` field of the global `rseq` structure; it should reset that field to `NULL` on exit. This setup implies that critical sections cannot nest, but these sections are meant to be short and should not be calling into other code anyway, so that will not be a problem. 

The code located at `abort_ip` must begin with the special `RSEQ_SIG` sentinel, which is defined in an architecture-dependent manner. Note that, if the abort code is invoked, the `rseq_cs` field will be zeroed by the kernel and must be assigned anew before reentering the critical section. 

There is also an `__rseq_flags` variable containing the flags that were used when registering with the kernel; according to [Weimer's documentation patch](<https://lwn.net/ml/libc-alpha/67665bb0d00cd540def5bfe758c94c98458536df.1638880889.git.fweimer@redhat.com/>), that variable is always set to zero for now. 

With that structure in place, applications using glibc can now use restartable sequences in a cooperative way. Unfortunately, there aren't really any useful examples of code using this new API to point to; this is all new stuff at this point. 

As readers have likely understood by now, actually coding the critical section almost certainly requires resorting to assembly language. This is clearly not a feature that is intended for casual or frequent use, but it can evidently produce significant performance gains in systems with high scalability requirements. Support in the GNU C Library will make restartable sequences a bit more accessible, but it seems destined to remain a niche feature used by few developers.
