---
title: Toward a safer fput()
url: https://lwn.net/Articles/494158/
date: "April 24, 2012"
category: "Filesystems-Virtual filesystem layer"
author: "By Jonathan Corbet April 24, 2012"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
April 24, 2012

Locking and the associated possibility of deadlocks are a challenge for developers working anywhere in the kernel. But that challenge appears to be especially acute in the virtual filesystem (VFS) layer, where the needs of many collaborating subsystems all come together in one place. The difficulties inherent in VFS locking were highlighted recently when the proposed [IMA appraisal extension](<https://lwn.net/Articles/488906/>) ran into review problems. The proposed fix shows that, while these issues can be worked around, the solution is not necessarily simple. 

The management of `file` structure reference counts is done with calls to `fget()` and `fput()`. A `file` structure, which represents an open file, can depend on a lot of resources: as long as a file is open, the kernel must maintain its underlying storage device, filesystem, network protocol information, security-related information, user-space notification requests, and more. An `fget()` call will ensure that all of those resources stay around as long as they are needed. A call to `fput()`, instead, might result in the destruction of any of those resources. For example, closing the last file on an unmounted filesystem will cause that filesystem to truly go away. 

What all this means is that a call to `fput()` can do a lot of work, and that work may require the acquisition of a number of locks. The problem is that `fput()` can also be called from any number of contexts; there are a few hundred `fput()` and `fput_light()` calls in the kernel. Each of those call sites has its own locking environment and, usually, no knowledge of what code in other subsystems may be called from `fput()`. So the potential for problems like locking-order violations is real. 

The IMA developers ran into exactly that sort of problem. The IMA appraisal cleanup code is one of those functions that can be invoked from an arbitrary `fput()` call. That code, it seems, sometimes needs to acquire the associated inode's `i_mutex` lock. But the locking rules say that, if both `i_mutex` and the task's `mmap_sem` are to be acquired, `i_mutex` must be taken first. If somebody calls `fput()` with `mmap_sem` held—something that happens in current kernels—the ordering rule will be violated, possibly deadlocking the system. A deadlocked system is arguably secure, but IMA users might be forgiven for complaining about this situation anyway. 

To get around this problem, IMA tried to check for the possibility of deadlock inside `fput()`, and, in that case, defer the underlying `__fput()` call (which does the real work) to a later and safer context. This idea [did not impress VFS maintainer Al Viro](<https://lwn.net/Articles/494173/>), who pointed out that there is no way to encode all of the kernel's locking rules into `fput()`. In such situations, it can be common for core kernel developers to say "NAK" and get back to what they were doing before, but Al continued to ponder the problem, saying: 

If it had been IMA alone, I would've cheerfully told them where to stuff the whole thing. Unfortunately, fput() *is* lock-heavy even without them. 

After thinking for a bit, he came up with [a plan](<https://lwn.net/Articles/494175/>) that offered a way out. Like the scheme used by IMA, Al's idea involves turning risky `fput()` calls into an asynchronous operation running in a separate thread. But there is no knowledge of locking rules added to `fput()`; instead, the situation is avoided altogether whenever possible, and all remaining calls are done asynchronously. 

In particular, Al is looking at all callers of `fget()` and `fput()` to see if they can be replaced with `fget_light()` and `fput_light()` instead. The "light" versions have a number of additional requirements: they come close to requiring that the calling code run in atomic context while the reference to the `file` structure is held. For a lot of situations - many system calls, for example - these rules don't get in the way. As the name suggests, the "light" versions are less expensive, so switching to them whenever possible makes sense regardless of any other issues. 

Then, `fput()` in its current form is renamed to `fput_nodefer()`. A new `fput()` is added that, when the final reference to a file is released, queues the real work to be done asynchronously later on. The "no defer" version will obviously be faster—the deferral mechanism will have a cost of its own—so its use will be preferred whenever possible. In this case, "whenever possible" means whenever the caller does not hold any locks. That is a constraint that can be independently verified for each call site; the "no defer" name should also hopefully serve as a warning for any developer who might change the locking environment in the future. 

With luck, all of the performance-critical calls can be moved to the "no defer" version, minimizing the performance hit that comes from the deferral of the `fput()` call. So it seems like a workable solution—except for [one little problem](<https://lwn.net/Articles/494181/>) pointed out by Linus: deferral can change the behavior seen by user space. In particular, the actual work of closing a file may not be complete by the time control returns to user space, causing the process's environment to differ in subtle and timing-dependent ways. Any program that expects that the cleanup work will be fully done by the time a `close()` call returns might just break. 

The "totally asynchronous deferral" literally *breaks*semantics*. 

Sure, it won't be noticeable in 99.99% of all cases, and I doubt you can trigger much of a test for it. But it's potential real breakage, and it's going to be hard to ever see. And then when it *does* happen, it's going to be totally impossible to debug. 

That does not seem like a good outcome either. The good news is that there is a potential solution out there in the form of Oleg Nesterov's [`task_work_add()`](<https://lwn.net/Articles/493429/>) patch set. This patch adds a functionality similar to workqueues, but with a fundamental difference: the work is run in the context of the process that was active at the time the work is added. 

In brief, the interface defines work to be done this way: 

```
#include <linux/task_work.h>
    
        typedef void (*task_work_func_t)(struct task_work *);
    
        struct task_work {
    	struct hlist_node hlist;
    	task_work_func_t func;
    	void *data;
        };
```

The `task_work` structure can be initialized with: 

```
void init_task_work(struct task_work *twork, task_work_func_t func, 
    		        void *data);
```

The work can be queued for execution with: 

```
int task_work_add(struct task_struct *task, struct task_work *twork, 
    		      bool notify);
```

A key aspect of this interface is that it will run any queued work before returning to user space from the kernel. So that work is guaranteed to be done before user space can run again; in the case of a function like `close()`, that guarantee means that user space will see the same semantics it did before, without subtle timing issues. So, Linus suggested, this API may be just what is needed to make the new `fput()` scheme work. 

There is just one final little problem: about a half-dozen architectures lack the proper infrastructure to support `task_work_add()` properly. That makes it unsuitable for use in the core VFS layer. Unless, of course, you're Al Viro; in that case it's just a matter of [quickly reviewing all the architectures](<https://lwn.net/Articles/494187/>) and coming up with a proposed fix—perhaps in assembly language—for each one. Assuming Al's work passes muster with the architecture maintainers, all of this work is likely to be merged for 3.5 - and the IMA appraisal work should be able to go in with it.
