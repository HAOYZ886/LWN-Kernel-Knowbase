---
title: "User-managed concurrency groups"
url: https://lwn.net/Articles/879398/
date: "December 28, 2021"
category: "Scheduler-User-managed concurrency groups; User-managed concurrency groups"
author: "By Jonathan Corbet December 28, 2021"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
December 28, 2021

The kernel's thread model is relatively straightforward and performs reasonably well, but that's not enough for all users. Specifically, there are use cases out there that benefit from a lightweight threading model that gives user space control over scheduling decisions. Back in May 2021, Peter Oskolkov posted a patch set implementing an abstraction known as [user-managed concurrency groups](<https://lwn.net/ml/linux-kernel/20210520183614.1227046-1-posk@google.com/>), or UMCG. Several revisions later, many observers still lack a clear idea of what this patch is supposed to do, much less whether it is a good idea for the kernel. Things have taken a turn, though, with Peter Zijlstra's [reimplementation of UMCG](<https://lwn.net/ml/linux-kernel/20211214204445.665580974@infradead.org/>). 

One developer reimplementing another's patch set is likely to raise eyebrows. Zijlstra's motivation for doing that work can perhaps be seen in [this message](<https://lwn.net/ml/linux-kernel/20211215222524.GH16608%40worktop.programming.kicks-ass.net/>), where he notes that the UMCG code looked little like the rest of the scheduler code. He also remarked that it required ""reverse engineering"" to figure out how UMCG was meant to be used. By the time that work was done, perhaps, it was just easier to recast the code in the form he thought it should take. 

In truth, the documentation for UMCG is no better than before — a significant problem for a major proposed addition to the system-call API. But it is possible to dig through the code (and a ""pretty rough"" [test application](<https://lwn.net/ml/linux-kernel/20211214210016.GD16608@worktop.programming.kicks-ass.net/>) posted by Zijlstra) to get a sense for what is going on. In short, UMCG calls for a multi-threaded application to divide itself into "server" and "worker" threads, where there is likely to be one server thread for each CPU on the system. Server threads make scheduling decisions, while workers run according to those decisions and get the actual work done. The advantage of a system like UMCG is that scheduling can happen quickly and with little overhead from the kernel — assuming the server threads are properly implemented, of course. 

#### Setting up

UMCG introduces three new system calls and one new structure that handles most of the communication with the kernel. Every thread participating in UMCG must have a `umcg_task` structure, which looks like this: 

```
struct umcg_task {
    	__u32	state;
    	__u32	next_tid;
    	__u32	server_tid;
    	__u64	runnable_workers_ptr;
    	/* ... */
        };
```

Some fields have been omitted here. Note that this structure, as it will eventually be provided by the C libraries, is likely to look different. The specific fields will be discussed as they become relevant. 

The first new system call is `umcg_ctl()`, which is used to register and unregister threads with the UMCG subsystem: 

```
int umcg_ctl(unsigned int flags, struct umcg_task *self, clockid_t which_clock);
```

The `flags` argument describes the operation to be performed, `self` is the `umcg_task` structure corresponding to the current thread, and `which_clock` controls the clock used for timestamps for this thread. 

If `flags` contains `UMCG_CTL_REGISTER`, then this call is registering a new thread with the subsystem. There are two alternatives, depending on which type of thread is being registered: 

  * If `flags` contains `UMCG_CTL_WORKER`, then this is a new worker task. In this case, `self->state` must be `UMCG_TASK_BLOCKED`, indicating that the worker is not initially running. The thread ID of the server that will handle this worker must be provided in `server_tid`. 
  * Otherwise, this is a server task. Its initial state must be `UMCG_TASK_RUNNING` (since it is indeed running) and `server_tid` must be the calling thread's ID. 

Workers and servers must be threads of the same process (more specifically, they must share the same address space). The system call returns zero if all goes well. For workers, though, that return will be delayed, as the calling thread will be blocked until the server schedules it to run. Registering a new worker will cause the indicated server to wake up. 

The other thing that happens when a worker is registered is that its state is set to `UMCG_TASK_RUNNABLE` and it is added to the server's singly-linked list of available workers. The list is implemented using the `runnable_workers_ptr` field in each task's `umcg_task` structure. The kernel will push a new task onto the head of the list with a compare-and-exchange operation; the server will normally use a similar operation to take tasks off the list. 

#### Scheduling

Most scheduling is done with calls to `umcg_wait()`: 

```
int umcg_wait(unsigned int flags, unsigned long timeout);
```

The `flags` field must be zero in the current patches. The calling thread must be registered as a UMCG thread or the call will fail. If the caller is a worker thread, the `timeout` must also be zero; this call will suspend execution of the worker and wake the associated server process for the next scheduling decision. If the worker's state is `UMCG_TASK_RUNNING` (as it should be if the task is running to make this call), it will be set back to `UMCG_TASK_RUNNABLE` and the task will be added to the server's `runnable_workers_ptr` list. Thus, for a worker task, a call to `umcg_wait()` is a way to yield to another thread while remaining runnable. 

In the case of the server, the usual reason for calling `umcg_wait()` is to schedule a new worker to run; this is done by setting the worker's thread ID in the `next_tid` field of the server's `umcg_task` structure before the call. If this is done, and the indicated thread is a UMCG worker in the `UMCG_TASK_RUNNABLE` state, it will be queued to run. The server, instead, will be blocked until either some sort of wakeup event happens or the specified `timeout` (if it is not zero) expires. 

One important detail is that the kernel, once it successfully wakes the new worker thread, will set the server's `next_tid` field to zero. That allows the server to quickly check, on return from `umcg_wait()`, whether the thread was actually scheduled or not. 

There are a few events that will cause a server to wake. If the current running worker blocks in a system call, for example, its state will be changed to `UMCG_TASK_BLOCKED`; the server can detect this by looking at the (previously) running worker's `umcg_task` structure. As noted above, a new task becoming runnable will cause a wakeup. If your editor's reading of the code is correct, there does not currently appear to be a way to notify the server that a worker task has exited entirely. 

#### Preemption

The `timeout` parameter to `umcg_wait()` can be used by server threads to implement forced preemption after a worker has run for a period of time. If `umcg_wait()` returns `ETIMEDOUT`, the server knows that the current worker has been running for the full timeout period; the server may then choose to make it surrender the CPU. That is done in a two-step process, the first of which is to add the `UMCG_TF_PREEMPT` flag to the running worker's `state` field (again, using a compare-and-exchange operation). Then a call should be made to the third new system call: 

```
int umcg_kick(unsigned int flags, pid_t tid);
```

Where `flags` must be zero and `tid` is the thread ID of the worker to be preempted. This call will cause the worker to re-enter the scheduler, at which point the `UMCG_TF_PREEMPT` flag will be noticed, the worker will be suspended, and it will be placed back onto the server's `runnable_workers_ptr` list. Once that completes, the server will wake again to schedule a new thread. 

That is pretty much the entirety of the new API at this point. This work is still clearly in an early state, though, and it would not be surprising to see a fair amount of evolution take place before it is considered for merging. UMCG arises out of Google's internal systems and reflects its use case, but there will almost certainly be other use cases for this sort of functionality, and those users have not yet made their needs known. As awareness of this work spreads, that situation can be expected to change. 

Oskolkov, meanwhile has, as one might expect, [required some convincing](<https://lwn.net/ml/linux-kernel/CAFTs51Xb6m=htpWsVk577n-h_pRCpqRcBg6-OhBav8OadikHkw@mail.gmail.com/>) that his work really needed to be rewritten by somebody else or that the new implementation is better. He [expressed](<https://lwn.net/ml/linux-kernel/CAPNVh5fenLG7uvdF1tjyfcOe8Ff3_L0-UqeCu9=tn-NMaJ3ikA@mail.gmail.com/>) discomfort with some of the changes, most notably Zijlstra's switch from a single queue of runnable workers to per-server queues. In the end, though, he said ""I'm OK with having it your way if all needed features are covered"". So it seems fair to assume that Zijlstra's patch reflects the future of this work. Time will tell where it goes from here.
