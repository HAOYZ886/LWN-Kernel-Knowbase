---
title: "LCA: A new approach to asynchronous I/O"
url: https://lwn.net/Articles/316806/
date: "January 27, 2009"
category: Asynchronous IO; Syslets
author: "By Jonathan Corbet January 27, 2009"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
January 27, 2009

Asynchronous I/O has been a problematic issue for the Linux kernel for many years. The current implementation is difficult to use, incomplete in its coverage, and hard to support within the kernel. More recently, there has been an attempt to resolve the problem with the [syslet concept](<http://lwn.net/Articles/236206/>), wherein kernel threads would be used to make almost any system call potentially asynchronous. Syslets have their own problems, though, not the least of which being that their use can cause a user-space process to change its process ID over time. Work on this area has slowed, with few updates being seen since mid-2007. 

Zach Brown is still working on the asynchronous I/O problem, though; he used his linux.conf.au talk to discuss his current approach. The new [![\[Zach
Brown\]](https://static.lwn.net/images/conf/lca2009/ZachBrown-sm.jpg)](<https://lwn.net/Articles/316844/>) "acall" interface has the potential to resolve many of the problems which have been seen in this area, but it is early-stage work which is likely to evolve somewhat before it is seriously considered for mainline inclusion. 

One of the big challenges with asynchronous kernel operations is that the kernel's idea of how to access task state is limited. For the most part, system calls expect the "`current`" variable to point to the relevant task structure. That proves to be a problem when things are running asynchronously, and, potentially, no longer have direct access to the originating process's state. The current AIO interface resolves this problem by splitting things into two phases: submission and execution. The submission phase has access to `current` and is able to block, but the execution phase is detached from all that. The end result is that AIO support requires a duplicate set of system call handlers and a separate I/O path. That, says Zach, is "why our AIO support still sucks after ten years of work." 

The fibril or syslet idea replaces that approach with one which is conceptually different: system call handlers remain synchronous, and kernel threads are used to add asynchronous operation on top. This work has taken the form of some tricky scheduler hacks; if an operation which is meant to be asynchronous blocks, the scheduler quickly shifts over to another thread and returns to user space in that thread. That allows the preservation of the state built up to the blocking point and it avoids the cost of bringing in a new thread if the operation never has to block. But these benefits at the cost of changing the calling process's ID - a change which is sure to cause confusion. 

When Zach inherited this work, he decided to take a fresh look at it with the explicit short-term goal of making it easy to implement the POSIX AIO specification. Other features, such as syslets (which allow a process to load a simple program into the kernel for asynchronous execution) can come later if it seems like a good idea. The end result is the "acall" API; this code has not yet been posted to the lists for review, but it is [available from Zach's web site](<http://oss.oracle.com/~zab/acall/>). 

With this interface, a user-space process specifies an asynchronous operation with a structure like this: 

```
struct acall_submission {
    	u32 nr;
    	u32 flags;
    	u64 cookie;
    	u64 completion_ring_pointer;
    	u64 completion_pointer;
    	u64 id_pointer;
    	u64 args[6];
        };
```

In this structure, `nr` identifies which system call is to be invoked asynchronously, while `args` is the list of arguments to pass to that system call. The `cookie` field is a value used by the calling program to identify the operation; it should be non-zero if it is to be used. The `flags` and various `_pointer` fields will be described shortly. 

To submit one or more asynchronous requests, the application will call: 

```
long acall_submit(struct acall_submission **submissions,
                          unsigned long nr);
```

`submissions` is a list of pointers to requests, and `nr` is the length of that list. The return value will be the number of operations actually submitted. If something goes wrong in the submission process, the current implementation will return a value less than `nr`, but the error code saying exactly what went wrong will be lost if any operations were submitted successfully. 

By default, `acall_submit()` will create a new kernel thread for each submitted operation. If the `flags` field for any request contains `ACALL_SUBMIT_THREAD_POOL`, that request will, instead, be submitted to a pool of waiting threads. Those threads are specific to the calling process, and they will only sit idle for 200ms before exiting. So submission to the thread pool may make sense if the application is submitting a steady stream of asynchronous operations; otherwise the kernel will still end up creating individual threads for each operation. Threads in the pool do not update their task state before each request, so they might be behind the current state of the calling process. 

If the `id_pointer` field is non-`NULL`, `acall_submit()` will treat it as a pointer to an `acall_id` structure: 

```
struct acall_id {
    	unsigned char opaque[16];
        };
```

This is a special value used by the application to identify this operation to the kernel. Internally it looks like this: 

```
struct acall_kernel_id {
    	u64 cpu;
    	u64 counter;
        };
```

It is, essentially, a key used to look up the operation in a red/black tree. 

The `completion_pointer` field, instead (if non-`NULL`), points to a structure like: 

```
struct acall_completion {
    	u64 return_code;
    	u64 cookie;
        };
```

The final status of the operation can be found in `return_code`, while `cookie` is the caller-supplied cookie value. Once that cookie has a non-zero value, the return code will be valid. 

The application can wait for the completion of specific operations with a call to: 

```
long acall_comp_pwait(struct acall_id **uids,
    			  unsigned long nr,
    			  struct timespec  *utime,
    			  const sigset_t *sigmask,
    			  size_t sigsetsize);
```

The `uids` array contains pointers to `acall_id` structures identifying the operations of interest; `nr` is the length of that array. If `utime` is not `NULL`, it points to a `timespec` structure specifying how long `acall_comp_pwait()` should wait before giving up. A set of signals to be masked during the operation can be given with `sigmask` and `sigsetsize`. A return value of one indicates that at least one operation actually completed. 

An application submitting vast numbers of asynchronous operations may want to avoid making another system call to get the status of completed operations. Such applications can set up one or more completion rings, into which the status of completed operations will be written. A completion ring looks like: 

```
struct acall_completion_ring {
    	uint32_t head;
    	uint32_t nr;
    	struct acall_completion comps[0];
        };
```

Initially, `head` should be zero, and `nr` should be the real length of the `comps` array. When the kernel is ready to store the results of an operation, it will first increment `head`, then put the results into `comps[head % nr]`. So a specific entry in the ring is only valid once the `cookie` field becomes non-zero. The kernel makes no attempt to avoid overwriting completion entries which have not yet been consumed by the application; it is assumed that the application will not submit more operations than will fit into a ring. 

The actual ring to use is indicated by the `completion_ring_pointer` value in the initial submission. Among other things, that means that different operations can go into different rings, or that the application can switch to a differently-sized ring at any time. In theory, it also means that multiple processes could use the same ring, though waiting for completion will not work properly in that case. 

If the application needs to wait until the ring contains at least one valid entry, it can call: 

```
long acall_ring_pwait(struct acall_completion_ring *ring,
    			  u32 tail, u32 min,
    			  struct timespec  *utime,
    			  const sigset_t *sigmask,
    			  size_t sigsetsize);
```

This call will wait until the given `ring` contains at least `min` events since the one written at index `tail`. The `utime`, `sigmask`, and `sigsetsize` arguments have the same meaning as with `acall_comp_pwait()`. 

Finally, an outstanding operation can be canceled with: 

```
long acall_cancel(struct acall_id *uid);
```

Cancellation works by sending a KILL signal to the thread executing the operation. Depending on what was being done, that could result in partial execution of the request. 

This API is probably subject to change in a number of ways. There is, for example, no limit to the size of the thread pool other than the general limit on the number of processes. Every request is assigned to a thread immediately, with threads created as needed; there is no way to queue a request until a thread becomes available in the future. The ability to load programs into the kernel for asynchronous execution ("syslets") could be added as well, though Zach gave the impression that he sees syslets as a relatively low-priority feature. 

Beyond the new API, this asynchronous operation implementation differs from its predecessors in a couple of ways. Requests will always be handed off to threads for execution; there is no concept of executing synchronously until something blocks. That may increase the overhead in cases where the request could have been satisfied without blocking, though the use of the thread pool should minimize that cost. But the big benefit is that the calling process no longer changes its ID when things do block. That results in a more straightforward user-space API with minimal surprises - certainly a good thing to do. 

Linus was at the presentation, and seemed to think that the proposed API was not completely unreasonable. So it may well be that, before too long, we'll see a version of the acall API proposed for the mainline. And that could lead to a proper solution to the asynchronous I/O problem at last.
