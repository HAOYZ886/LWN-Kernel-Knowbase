---
title: Alternatives to fibrils
url: https://lwn.net/Articles/221913/
date: "February 13, 2007"
category: Fibrils; Syslets
author: 
---

Since the writing of [last week's article on fibrils](<http://lwn.net/Articles/220897/>), there has been relatively little discussion of that set of patches. That silence does not mean that interest in the idea has faded for now, however; instead, a couple of different approaches have been posted for consideration. 

Linus Torvalds got inspired to create [an asynchronous system call patch](<https://lwn.net/Articles/221914/>) of his own. Simplicity is the word to describe this patch: it adds less than 200 lines of code to the kernel (""I even added comments, so a lot of the few new added lines aren't even code!""). It works like this: 

  * The new `async()` system call takes a system call number, arguments for the system call, and a pointer to a location for the final status code. 

  * The process's register set is saved, then the system call is executed as usual. 

  * Should the kernel call `schedule()`, meaning that the system call is about to block, the process will fork before blocking. 

  * The new _child_ process returns to user space and continues executing there. Meanwhile, the original process will finish out the asynchronous system call. 

The largest claimed advantage to this patch, beyond its simplicity, is that there is almost no overhead if the asynchronous system call can be completed without blocking. The fibril patch, instead, always runs asynchronous calls in independent fibrils. Linus claims that almost all asynchronous system calls can, in fact, be completed synchronously without blocking, so he would really rather see little or no up-front cost in that case. 

There are various issues with Linus's patch. If the asynchronous call blocks, for example, the return to user space will happen in a different process - a change which could prove confusing to user space. Only one asynchronous operation can be outstanding at any given time. There is also no way to wait for an asynchronous operation to complete except to poll the exit status. But this patch was never meant to be a complete solution; as a proof of concept it is interesting. 

For a rather more elaborate approach, Ingo Molnar's [syslet patchset](<http://lwn.net/Articles/221887/>) is worth a look. With syslets, a user-space program can run system calls asynchronously. Beyond that, however, it can load little programs into the kernel and let them run independently. 

To use syslets, the application starts by filling in one of these structures: 

```
struct syslet_uatom {
    	unsigned long		flags;
    	unsigned long		nr;
    	long 	 		*ret_ptr;
    	struct syslet_uatom	*next;
    	unsigned long		*arg_ptr[6];
    	void 	 		*private;
        };
```

Here, `nr` is the number of the system call to run, `arg_ptr` holds pointers to the arguments, and `ret_ptr` tells the kernel where to put the final status from the call. The `private` field is not used by the kernel at all. We'll get to the other fields shortly. 

Once the `syslet_uatom` structure is ready, the application can run it with: 

```
long async_exec(struct syslet_uatom *atom);
```

This call will start on the requested system call immediately. If that system call never blocks, it will be run synchronously and the address of the `atom` will be returned from `async_exec()`. Otherwise the kernel will grab a thread from a pool and use that thread to return to user space, continuing the system call in the original thread. The application can then go off and do whatever makes sense - including running more syslets - while the system call runs to completion. 

What actually happens when the system call completes is a little more complex and interesting, however. Unless user space has requested otherwise, the kernel does _not_ just complete the syslet after the first system call runs; instead, it looks at the `next` field of the `syslet_uatom` structure. If that field is non-`NULL`, it is taken as the user-space address of the next syslet to be run by the kernel. In other words, an application is not restricted to running individual asynchronous system calls; it can chain up a whole series of them to run without ever exiting the kernel. The cost of fetching a new syslet atom is far less than a transition to user space and back, so there is a significant performance improvement to be had just by chaining two system calls together. 

The final field in `struct syslet_uatom` is `flags`, which controls how syslets are executed. Four of them (`SYSLET_STOP_ON_NONZERO`, `SYSLET_STOP_ON_ZERO`, `SYSLET_STOP_ON_NEGATIVE`, and `SYSLET_STOP_ON_NON_POSITIVE`) will test the result of the current atom's system call and, possibly, terminate execution of the syslet. In this way, for example, a chain of system calls can be stopped early if one of them fails. It is also possible to create a kernel-space loop which reads a file until no more data is available. 

The `SYSLET_SKIP_TO_NEXT_ON_STOP` modifies the above flags so that, rather than terminating the syslet, the kernel skips to an atom found immediately after the current one in the process's address space. This flag allows a syslet to terminate a loop and move on to further processing within the syslet. If an application knows that a syslet will block, it can request asynchronous execution from the outset with `SYSLET_ASYNC`. There is also a `SYSLET_SYNC` flag which causes the whole thing to run synchronously. 

Syslets do not have any variables of their own. To help with the writing of useful programs, Ingo has added a new system call: 

```
long umem_add(unsigned long *pointer, unsigned long increment);
```

This call simply adds the given `increment` to `*pointer`, returning the resulting value. 

The application can register a ring buffer with the kernel using the `async_register()` system call. Whenever an atom completes, its address will be stored in the next ring buffer entry; the application can then use that address to find the system call status. The kernel will not overwrite non-`NULL` ring buffer entries, so the application must reset them as it consumes them. If the application needs to wait for syslet completion, it can call: 

```
long async_wait(unsigned long min_events);
```

This call will block the process until at least `min_events` have been stored into the ring buffer. 

This patch set, too, presents a number of unanswered questions. Once again, signal handling has been punted for now. There's no end of security implications which must be thought out; in the end, a number of system calls will probably be marked as being off-limits for asynchronous execution. There has still been no discussion on how this sort of interface would play with the kevent patches - kevents seem to be concept that nobody wants to talk about at the moment. 64/32-bit compatibility could present interesting challenges of its own. And so on. But the initial reaction to syslets appears to be positive (though [Linus hates it](<https://lwn.net/Articles/222168/>)); syslets might just point to the form of the fibril idea which eventually makes it into the mainline kernel.
