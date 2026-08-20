---
title: Threadlets
url: https://lwn.net/Articles/223899/
date: "February 27, 2007"
category: Asynchronous IO; Syslets; Threadlets
author: 
---

Remember [fibrils](<http://lwn.net/Articles/219954/>)? The memory may be dim, seeing as the fibril concept was posted way back in January, but the work inspired by this idea continues. The latest [syslet patch](<http://lwn.net/Articles/223800/>) from Ingo Molnar was posted on February 24; it brings some interesting changes to this approach to asynchronous system call execution. 

The concept of "atoms" which was part of [the first syslet patch](<http://lwn.net/Articles/221913/>) remains; an atom is a unit of work which is executed in kernel space. Atoms can be chained together with some simple flow control operations, with the entire sequence being executed without leaving the kernel. A sequence of atoms will be executed synchronously if possible; if an atom blocks, however, a new thread will be created to return to user space. As a result, asynchronous code can be executed in parallel, but the overhead of thread creation is only incurred when there is a need for it. 

The syslet API has changed, however, in response to some concerns about how completion events were handled. User space must now create create a structure to go along with the atom sequence: 

```
struct async_head_user {
    	unsigned long				kernel_ring_idx;
    	unsigned long				user_ring_idx;
    	struct syslet_uatom __user		**completion_ring;
    	unsigned long				ring_size_bytes;
    	/* There is other stuff here too */
        };
```

This structure defines the completion ring - a circular buffer which is filled (by the kernel) with pointers to atoms which have completed execution. There is no longer a need to register this buffer with the kernel; instead, the structure is passed in when the atoms are passed to the kernel for execution: 

```
struct syslet_uatom *async_exec (struct syslet_uatom *atom,
                                         struct async_head_user *ahu);
```

An implication of this new interface is that each chain of atoms can, if desired, have its own completion ring. These rings are no longer pinned into memory, so there can be an arbitrary number of them. The return value from `async_exec()` will be a pointer to the last atom to execute if the chain runs without blocking, or `NULL` if the chain blocked and user space is running in a new thread. 

Jens Axboe, Suparna Bhattacharya, and others have been doing some benchmarking with the current syslet code. Many (but not all) of the benchmark runs show that syslets perform better than the current asynchronous I/O implementation. The causes for the divergence between results are still being investigated; one thing that has come out is that the CFQ I/O scheduler does not work properly with syslets. CFQ takes a process-oriented approach to scheduling, so it is not entirely surprising that changes to the process model could prove confusing there. Nonetheless, Ingo [is confident](<https://lwn.net/Articles/223925/>) that syslets are a performance win: 

[I]n my own (FIO based) measurements syslets beat the native KAIO interfaces both in the cached and in the non-cached [== many threads] case. I did not expect the latter at all: the non-cached syslet codepath is not optimized at all yet, so i expected it to have (much) higher CPU overhead than KAIO. 

This means that KAIO is in worse shape than i thought - there's just way too much context KAIO has to build up to submit parallel IO contexts. Many years of optimizations went into KAIO already, so it's probably at its outer edge of performance capabilities. 

Perhaps the biggest change in the new patch set, however, is the creation of a new concept known as "threadlets." The threadlet idea brings the on-demand thread creation idea to user space. Threadlets are ordinary user-space code which will be run synchronously if possible; should this code block, however, a new thread will be created to allow user space to continue while the threadlet waits. 

The API as described by Ingo requires the application to define a function to run as a threadlet: 

```
long threadlet_fn(void *data)
        {
            /* Almost anything can go here */
    	return complete_threadlet_fn(event, ahu);
        }
```

About the only thing which is different here is that the call to `complete_threadlet_fn()` is required: 

```
long complete_threadlet_fn(void *event, struct async_head_user *ahu);
```

The `event` parameter is stored in the completion ring - since there is no atom structure here, user-space must provide a value to identify which threadlet completed. The `async_head_user` structure describes the completion ring, as before. 

The application can fire off a threadlet with: 

```
long threadlet_exec(long threadlet_fn(void *),
                            unsigned long stack,
    			struct async_user_head *ahu);
```

Besides the `threadlet_fn()` described above, this call requires that the application provide stack space for the new threadlet. The `stack` argument is thus a pointer (despite its `unsigned long` type) to a few pages of ordinary user-space memory set aside for this purpose. There is also an `async_user_head` structure to provide for the reporting of threadlet completion. If `threadlet_fn()` runs to completion without blocking, the return value of `threadlet_exec()` will be `1`; otherwise zero is returned. 

As it happens, `threadlet_exec()` is a user-space wrapper which hides much of the complexity of the real interface. This function switches over to the given `stack` immediately, then calls `threadlet_on()`, which is a true system call, passing it the original stack address as a parameter. This call saves that stack address, ensures that a "cache miss thread" will be available if needed, and marks the process as running in an asynchronous mode. It then returns to user space, which executes the user's `threadlet_fn()`. Should that function block, the kernel will grab a new thread, set it up with the original stack, and send it back to user space. The threadlet function will then continue to execute in the original thread once the condition which blocked it is resolved. 

Unsurprisingly, `complete_threadlet_fn()` is also a wrapper. It calls `threadlet_off()` to indicate that the execution of the threadlet is complete. If `threadlet_off()` returns `1`, the threadlet ran synchronously and there is no more to do. Otherwise, a call is made to: 

```
long async_thread(void *event, struct async_head_user *ahu);
```

This system call will store `event` in the completion ring. Since this thread is running asynchronously, returning to user space is not in the cards - user space went its own way when things first blocked. So `async_thread()` puts the current thread onto the list of threads available the next time one is needed for asynchronous execution. 

The above description has left out a couple of details, mostly related to the management of user-space stacks. It's worth noting that there appears to be no guard page put at the end of a threadlet stack, meaning that, if the stack is too small, user space could easily overflow it. The result would likely be some truly obscure bugs which would not be fun to find. This API could also change a bit; Ingo apparently has plans for turning `threadlet_on()` and `threadlet_off()` into vsyscalls which could execute without going into the kernel at all. That, of course, would improve the performance of threadlets further. 

While the syslet interface provided interesting functionality, it was immediately seen as being hard to work with. The new threadlet API was designed to get around those objections by getting away from the whole "atom" concept and making it possible to run user-space code asynchronously with a minimum of fuss. The syslet mechanism is likely to remain, as it will still be the fastest way to get a task done. But syslets may see little use outside of special-purpose libraries which hide their complexity. For everything else, threadlets could prove to be the way to go.
