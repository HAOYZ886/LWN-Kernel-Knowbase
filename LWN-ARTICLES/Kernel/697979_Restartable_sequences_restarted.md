---
title: Restartable sequences restarted
url: https://lwn.net/Articles/697979/
date: "August 24, 2016"
category: Restartable sequences
author: "By Jonathan Corbet August 24, 2016"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
August 24, 2016

"Restartable sequences" is starting to look a bit like one of those bright ideas that floats around on the kernel list for years, but which never quite seems to make it into the mainline. In this case, the idea was [first proposed](<https://lwn.net/Articles/650333/>) over one year ago without, yet, having made appreciable progress toward merging; activity on this patch set died down after a while. But development on restartable sequences has picked up again under a new developer who has come up with yet another API for the feature. 

As has happened in the kernel, scalability pressures are driving some user-space applications toward the use of lockless algorithms. In kernel space, such algorithms tend to be based on either disabling preemption or retrying an operation after contention is detected. Disabling preemption in user space is not an option, so retries are the primary option remaining. That is where restartable sequences come in; they combine a kernel-facilitated mechanism for detecting possible contention with a means to quickly force a retry when contention happens. 

The current version of restartable sequences, as [posted](<https://lwn.net/Articles/697756/>) by Mathieu Desnoyers, retains the core idea of its predecessors. A restartable sequence is based around a short segment of code; only the final instruction of that segment is allowed to have side effects visible outside of the current thread. There is also an abort sequence, called to clean up and retry should the thread be preempted while executing the sequence. The specifics have changed, though. 

Code using restartable sequences needs to start with an `rseq` structure: 

```
struct rseq {
        	int32_t cpu_id;
    	uint32_t event_counter;
            struct rseq_cs *rseq_cs;
        };
```

(The actual structure is a bit more complex; various architecture-specific details have been omitted here in the interest of readability.) The `cpu_id` field always contains the number of the CPU on which the thread is running; `event_counter` is incremented whenever the thread is preempted — but only if `rseq_cs` is not null. The purpose of `rseq_cs` will be discussed below. 

This structure must be registered with the kernel before restartable sequences can be used; the operative system call is: 

```
int rseq(struct rseq *rseq, int flags);
```

Only one `rseq` structure can be registered at a time in any given thread, but that structure can be registered multiple times, and the kernel will keep track of how many registrations (and unregistrations) there have been. The `flags` argument must be zero when registering a new structure. Unregistration is done by passing a null pointer for the `rseq` structure; setting `flags` to `RSEQ_FORCE_UNREGISTER` will cause the immediate removal of the structure, even if it has been registered multiple times. 

In the past there have been concerns about how the restartable sequences feature would work when there are multiple users within an application (libraries, for example) that do not know about each other. If those users fight over which `rseq` structure is used, there will be problems with this interface as well; if, instead, they can all agree on the same structure, all will be well. Restartable sequences must be simple, so it makes no sense for code running within one to call another function at all, much less one that would start its own sequence. So there can only be a single sequence running at any given time. 

To ensure that all users share a single `rseq` structure, the documentation recommends that each user declare it as a weak symbol and name it `__rseq_abi`. The linker will then ensure that, if there are multiple declarations within a given program, they will all refer to the same structure. 

The other half of the puzzle is the `rseq_cs` structure pointed to from within the `rseq` structure above. This structure looks like (again, with some simplification applied): 

```
struct rseq_cs {
            void *start_ip;
    	void *post_commit_ip;
    	void *abort_ip;
        };
```

This structure describes an actual critical section that runs in the restartable mode. Here, `start_ip` is the address of the first instruction in the section, and `post_commit_ip` is the first instruction beyond the end of the section; any code running between those two instructions is running within the critical section. The `abort_ip` pointer is the address of the cleanup code to be executed should the thread be preempted while executing within the section. 

With those pieces, a restartable sequence is run using something like this sequence of steps (assuming that the `rseq` structure is already registered): 

  1. The `event_counter` field from the `rseq` structure is read and saved. 
  2. The `rseq_cs` pointer in the `rseq` structure is set to point to the `rseq_cs` structure describing the critical section to be executed. 
  3. The `event_counter` is read again and compared to the value read previously; if the values do not match, the `rseq_cs` field should be cleared and the process must be restarted from the beginning. 
  4. The critical section can now be executed. In most cases, only the final instruction in the critical section should have visible side effects. 
  5. The `rseq_cs` field should be set to `NULL`. 

If execution makes it past the end of the section, then all is well. If, instead, the thread is preempted while running within the critical section, the kernel will cause it to jump to the `abort_ip` address. The code found there should clean up and prepare to retry. 

In principle, that is all there is to it. In practice, applications using this feature must still include some assembly code to set up the various instruction pointers; there is some complexity involved in making it all work properly. Those interested in examples can have a look at [the self-tests](<https://lwn.net/Articles/697990/>) included with the patch and, in particular, the rather frightening assembly-in-CPP code found [here](<https://lwn.net/Articles/697990/#do_rseq>) and [here](<https://lwn.net/Articles/697990/#finish_asm>). 

There have not been many comments on the implementation this time around; it seems that, perhaps, things are finally getting to a point where the developers who are paying attention are reasonably happy. The next obstacle, though, may be Linus, who [wants more evidence](<https://lwn.net/Articles/697991/>) that this is a feature that will actually be used. Convincing him is likely to require demonstrating some real-world code that benefits from the feature and benchmarks to prove that it is all worthwhile. Since restartable sequences are said to have been in use in places like Google for some time, that proof should be possible to come by. If the developers involved follow through, perhaps this sequence of patches will not need to be restarted too many more times.
