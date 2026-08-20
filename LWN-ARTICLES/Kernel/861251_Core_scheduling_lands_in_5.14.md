---
title: Core scheduling lands in 5.14
url: https://lwn.net/Articles/861251/
date: "July 1, 2021"
category: "Releases-5.14; Scheduler-Core scheduling"
author: "By Jonathan Corbet July 1, 2021"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
July 1, 2021

The [core scheduling](<https://lwn.net/Articles/780703/>) feature has been under discussion for over three years. For those who need it, the wait is over at last; core scheduling was merged for the 5.14 kernel release. Now that this work has reached a (presumably) final form, a look at why this feature makes sense and how it works is warranted. Core scheduling is not for everybody, but it may prove to be quite useful for some user communities. 

Simultaneous multithreading (SMT, or "hyperthreading") is a hardware feature that implements two or more threads of execution in a single processor, essentially causing one CPU to look like a set of "sibling" CPUs. When one sibling is executing, the other must wait. SMT is useful because CPUs often go idle while waiting for events — usually the arrival of data from memory. While one CPU waits, the other can be executing. SMT does not result in a performance gain for all workloads, but it is a significant improvement for most. 

SMT siblings share almost all of the hardware in the CPU, including the many caches that CPUs maintain. That opens up the possibility that one CPU could extract data from the other by watching for visible changes in the caches; the Spectre class of hardware vulnerabilities have made this problem far worse, and there is little to be done about it. About the only way to safely run processes that don't trust each other (with current kernels) is to disable SMT entirely; that is a prospect that makes a lot of people, cloud-computing providers in particular, distinctly grumpy. 

While one might argue that cloud-computing providers are usually grumpy anyway, there is still value in anything that might improve their mood. One possibility would be a way to allow them to enable SMT on their systems without opening up the possibility that their customers may use it to attack each other; that could be done by ensuring that mutually distrusting processes do not run simultaneously in siblings of the same CPU core. Cloud customers often have numerous processes running; spamming Internet users at scale requires a lot of parallel activity, after all. If those processes can be segregated so that all siblings of any given core run processes from the same customer, we can be spared the gruesome prospect of one spammer stealing another's target list — or somebody else's private keys. 

Core scheduling can provide this segregation. In abstract terms, each process is assigned a "cookie" that identifies it in some way; one approach might be to give each user a unique cookie. The scheduler then enforces a regime where processes can share an SMT core only if they have the same cookie value — only if they trust each other, in other words. 

More specifically, core scheduling is managed with the [`prctl()`](<https://man7.org/linux/man-pages/man2/prctl.2.html>) system call, which is defined generically as: 

```
int prctl(int option, unsigned long arg2, unsigned long arg3,
                  unsigned long arg4, unsigned long arg5);
```

For core-scheduling operations, `option` is `PR_SCHED_CORE`, and the rest of the arguments are defined this way: 

```
int prctl(PR_SCHED_CORE, int cs_command, pid_t pid, enum pid_type type,
    	      unsigned long *cookie);
```

There are four possible operations that can be selected with `cs_command`: 

  * `PR_SCHED_CORE_CREATE` causes the kernel to create a new cookie value and assign it to the process identified by `pid`. The `type` argument controls how widely spread this assignment is; `PIDTYPE_PID` only changes the identified process, for example, while `PIDTYPE_TGID` assigns the cookie to the entire thread group. The `cookie` argument must be NULL. 
  * `PR_SCHED_CORE_GET` retrieves the cookie value for `pid`, storing it in `cookie`. Note that there is not much that a user-space process can actually do with a cookie value; its utility is limited to checking whether two processes have the same cookie. 
  * `PR_SCHED_CORE_SHARE_TO` assigns the calling process's cookie value to `pid` (using `type` to control the scope as described above). 
  * `PR_SCHED_CORE_SHARE_FROM` fetches the cookie from `pid` and assigns it to the calling process. 

Naturally, a process cannot just fetch and assign cookies at will; the usual "can this process call [`ptrace()`](<https://man7.org/linux/man-pages/man2/ptrace.2.html>) on the target" test applies. It is also not possible to generate cookie values in user space, a restriction that is necessary to ensure that unrelated processes get unique cookie values. By only allowing cookie values to propagate between processes that already have a degree of mutual trust, the kernel prevents a hostile process from setting its own cookie to match that of a target process. 

Whenever a CPU enters the scheduler, the highest-priority task will be picked to run in the usual way. If core scheduling is in use, though, the next step will be to send an inter-processor interrupt to the sibling CPUs, each of which will respond by checking the newly scheduled process's cookie value against the value for the process running locally. If need be, the interrupted processor(s) will switch to running a process with an equal cookie, even if the currently running process has a higher priority. If no compatible process exists, the processor will simply go idle until the situation changes. The scheduler will migrate processes between cores to prevent the forced idling if possible. 

Early versions of the core-scheduling code had a significant throughput cost for the system as a whole; indeed, it was sometimes worse than just disabling SMT altogether, which rather defeated the purpose. The code has been through a number of revisions since then, though, and apparently performs better now. There will always be a cost, though, to a mechanism that will occasionally force processors to go idle when runnable processes exist. For that reason core scheduling, as Linus Torvalds [put it](<https://lwn.net/ml/linux-kernel/CAHk-=wjNP8Oi4nve=uu=Q3+rGar3CY9fBUQFuJK-WYCK3F198w@mail.gmail.com/>), ""makes little sense to most people"". It can be beneficial, though, in situations where the only alternative is to turn off SMT completely. 

While the security use case is driving the development of core scheduling, [there are other use cases](<https://lwn.net/Articles/799454/>) as well. For example, systems running realtime processes usually must have SMT disabled; you cannot make any response-time guarantees when the CPU has to compete with a sibling for the hardware. Core scheduling can ensure that realtime processes get a core to themselves while allowing the rest of the system to use SMT. There are other situations where the ability to control the mixing of processes on the same core can bring benefits as well. 

So, while core scheduling is probably not useful for most Linux users, there are user communities that will be glad that this feature has finally found its way into the mainline. Adding this sort of complication to a central, performance-critical component like the scheduler was never going to be easy but, where there is sufficient determination, a way can be found. The developers involved have certainly earned a cookie for pushing this work to a successful completion.
