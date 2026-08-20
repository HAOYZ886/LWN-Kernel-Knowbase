---
title: Restartable sequences
url: https://lwn.net/Articles/650333/
date: "July 7, 2015"
category: Restartable sequences
author: "By Jonathan Corbet July 7, 2015"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
July 7, 2015

Concurrent code running in user space is subject to almost all of the same constraints as code running in the kernel. One of those is that cross-CPU operations tend to ruin performance, meaning that data access should be done on a per-CPU basis whenever possible. Unlike kernel code, though, user-space per-CPU code cannot enter atomic context; it, thus, cannot protect itself from being preempted or moved to another CPU. The [restartable sequences patch set](<https://lwn.net/Articles/649288/>) recently posted by Paul Turner demonstrates one possible solution to that problem by providing a limited sort of atomic context for user-space code. 

Imagine maintaining a per-CPU linked list, and needing to insert a new element at the head of that list. Code to do so might look something like this: 

```
new_item->next = list_head[cpu];
        list_head[cpu] = new_item;
```

Such code faces a couple of hazards in a multiprocessing environment. If it is preempted between the two statements above, another process might slip in and insert its own new element; when the original process resumes, it will overwrite `list_head[cpu]`, causing the loss of the item added while it was preempted. If, instead, the process is moved to a different CPU, it could get confused between each CPU's list or run concurrently with a new process on the original CPU; the result in either case would be a corrupted list and late-night phone calls to the developer. 

These situations are easily avoidable by using locks, but locks are expensive even in the absence of contention. The same holds for atomic operations like compare-and-swap; they work, but the result can be unacceptably slow. So developers have long looked for faster alternatives. 

The key observation behind restartable sequences is that the above code shares a specific feature with many other high-performance critical sections, in that it can be divided into two parts: (1) an arbitrary amount of setup work that can be thrown away and redone if need be, and (2) a single instruction that "commits" the operation. The first line in that sequence: 

```
new_item->next = list_head[cpu];
```

has no visible effect outside the process it is executing in; if that process were preempted after that line, it could just execute it again and all would be well. The second line, though: 

```
list_head[cpu] = new_item;
```

has effects that are visible to any other process that uses the list head. If the executing process has been preempted or moved in the middle of the sequence, that last line must not be executed lest it corrupt the list. If, instead, the sequence has run uninterrupted, this assignment can be executed with no need for locks or atomic instructions. That, in turn, would make it fast. 

A restartable sequence as implemented by Paul's patch is really just a small bit of code stored in a special region of memory; that code implements both the setup and commit stages as described above. If the kernel preempts a process (or moves it to another CPU) while the process is running in that special section, control will jump to a special restart handler. That handler does whatever is needed to restart the sequence; often (as it would be in the linked-list case) it's just a matter of going back to the beginning and starting over. 

The sequence must adhere to some restrictions; in particular, the commit operation must be a single instruction and code within the special section cannot invoke any code outside of it. But, if it holds to the rules, a repeatable sequence can function as a small critical section without the need for locks or atomic operations. In a sense, restartable sequences can be thought as a sort of poor developer's transactional memory. If the operation is interrupted before it commits, the work done so far is simply tossed out and it all restarts from the beginning. 

Paul's patch adds a new system call: 

```
int restartable_sequences(int op, int flags, long val1, long val2, long val3);
```

There are two operations that can be passed as the `op` parameter: 

  * `SYS_RSEQ_SET_CRITICAL` sets the critical region; `val1` and `val2` are the bounds of that region, and `val3` is a pointer to the restart handler (which must be outside of the region). 

  * `SYS_RSEQ_SET_CPU_POINTER` specifies a location (in `val1`) of an integer variable to hold the current CPU number. This location should be in thread-local storage; it allows each thread to quickly determine which CPU it is running on at any time. 

The CPU-number pointer is needed so that each section can quickly get to the correct per-CPU data; to emphasize that, the restart handler will not actually be called until this pointer has been set. Only one region for restartable sequences can be established (but it can contain multiple sequences if the restart handler is smart enough), and the region is shared across all threads in a process. 

Paul notes that Google is using this code internally now; it was also [discussed at the Linux Plumbers Conference [PDF]](<http://www.linuxplumbersconf.org/2013/ocw/system/presentations/1695/original/LPC%20-%20PerCpu%20Atomics.pdf>) in 2013. He does not believe it is suitable for mainline inclusion in its current form, though. The single-region limitation does not play well with library code, the critical section must currently be written in assembly, and the interactions with thread-local storage are painful. But, he thinks, it is a reasonable starting place for a discussion on how a proper interface might be designed. 

Paul's patch is not the only one in this area; Mathieu Desnoyers posted [a patch set with similar goals](<https://lwn.net/Articles/645717/>) back in May. Given [Linus's reaction](<http://thread.gmane.org/gmane.linux.kernel/1959603/focus=1961389>), it's safe to say that Mathieu's patch will not be merged anytime soon, but Mathieu did achieve [his secondary goal](<http://thread.gmane.org/gmane.linux.kernel/1959603/focus=1962052>) of getting Paul to post his patches. In any case, there is clearly interest in mechanisms that can improve the performance of highly concurrent user-space code, so we will almost certainly see more patches along these lines in the future.
