---
title: "process_madvise(), pidfd capabilities, and the revenge of the PIDs"
url: https://lwn.net/Articles/810076/
date: "January 21, 2020"
category: Memory management; System calls; pidfd
author: "By Jonathan Corbet January 21, 2020"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
January 21, 2020

Once upon a time, there were few ways for one process to operate upon another after its creation; sending signals and `ptrace()` were about it. In recent years, interest in providing ways for processes to control others has been on the increase, and the kernel's process-management API has been expanded accordingly. Along these lines, the `process_madvise()` system call has been [proposed](<https://lwn.net/Articles/789153/>) as a way for one process to influence how memory management is done in another. There is a new `process_madvise()` series which is interesting in its own right, but this series has also raised a couple of questions about how process management should be improved in general. 

The existing [`madvise()`](<http://man7.org/linux/man-pages/man2/madvise.2.html>) system call allows a process to make suggestions to the kernel about how its address space should be managed. The 5.4 kernel saw a couple of new types of advice that could be provided with `madvise()`: `MADV_COLD` and `MADV_PAGEOUT`. The former requests that the kernel place the indicated range of pages onto the inactive list, essentially saying that they have not been used in a long time. Those pages will thus be among the first considered for reclaim if the kernel needs memory for other purposes. `MADV_PAGEOUT`, instead, is a stronger statement that the indicated pages are no longer needed; it will cause them to be reclaimed immediately. 

These new requests are useful for processes that know what their future access patterns will be. But it seems that in certain environments — Android, in particular — processes lack that knowledge, but the management system does know when certain memory ranges are no longer needed. The bulk of a process's address space could be marked as `MADV_COLD` when that process is moved out of the foreground, for example. In such settings, letting one process call `madvise()` on behalf of another helps the system as a whole make the best use of its memory resources. That is the purpose behind the `process_madvise()` proposal. 

When Minchan Kim [posted](<https://lwn.net/ml/linux-kernel/20200110213433.94739-3-minchan@kernel.org/>) a new `process_madvise()` series in early January, the API looked like this: 

```
int process_madvise(int pidfd, void *addr, size_t length, int advice,
        			unsigned long flags);
```

The effect of this call would be the same as if the process identified by `pidfd` (which, as one might guess from the name, is a [pidfd](<https://lwn.net/Articles/794707/>) rather than a process ID) had called `madvise()` with the given `addr`, `length`, and `advice` arguments; the `flags` argument is not currently used. Only a subset of `madvise()` comments (`MADV_COLD`, `MADV_PAGEOUT`, `MADV_MERGEABLE`, and `MADV_UNMERGEABLE`) is supported; others can be added as the use cases for them emerge. 

It seemed like most of the discussion on `process_madvise()` had run its course by now, and that there were few obstacles to its path into the mainline kernel. But, then, a couple of issues came up. 

#### Pidfd capabilities

One of those doesn't directly affect `process_madvise()`, but does offer an interesting look into where the pidfd API is headed in general. In the current patch sets, the question of whether one process is allowed to call `process_madvise()` on another is answered with the usual "would `ptrace()` be allowed?" test. That comes down to either running under the same user ID or having the `CAP_SYS_PTRACE` capability. This test is standard practice and uncontroversial, but Christian Brauner, the developer behind most of the pidfd work, [has another idea](<https://lwn.net/ml/linux-kernel/20200113191046.2tidyvc544zvchek@wittgenstein/>) in mind: 

When you create a pidfd, e.g. with clone3() and you'd wanted to use it for madvise you'd need to set a flag like pidfd_cap_madvise or pidfd_feature_madvise when you create the pidfd. Only if the pidfd was created with that flag set could you use it with madvise. 

In essence, under this scheme, a pidfd would include a capability mask of its own stating what could be done with it. It is analogous to how ordinary file descriptors work: one can only write to a file descriptor if it was created with a flag that allows writing. Brauner [requested](<https://lwn.net/ml/linux-kernel/20200113104256.5ujbplyec2sk4onn@wittgenstein/>) that `process_madvise()` not be merged until 5.7 so that it could use this (not yet implemented) capability mechanism from the beginning. 

There is an interesting question about how pidfd capabilities would work, [brought up](<https://lwn.net/ml/linux-kernel/CAKOZuev5k3EquMd-6VbvruahjjtxQzRhUVo2ttgVyk+yYz9aOA@mail.gmail.com/>) by Daniel Colascione: would having a pidfd with a given capability flag be sufficient to carry out a specific action, or would the traditional tests apply too? In other words, if a process holds a pidfd that was created with the ability to pass it to `process_madvise()`, would that process still need to pass the `ptrace()` test for the action to succeed? 

Brauner [admitted](<https://lwn.net/ml/linux-kernel/20200113204237.ew6nn4ohxu7auw3u@wittgenstein/>) to ""going back and forth"" on that question; he said, though, that his inclination was to still require the privilege to execute the operation independently from the capability flag on the pidfd. So those capabilities (or the lack thereof) would serve only to restrict operations that would otherwise be allowed. Colascione [argued](<https://lwn.net/ml/linux-kernel/CAKOZueu=U4c2URaq8Pz-B00XV+TxaKwHRNXv3BUiDbQrLQpJ3A@mail.gmail.com/>) in favor of pidfd capabilities actually granting access, though, and Brauner [agreed](<https://lwn.net/ml/linux-kernel/20200114192056.b6wi4adsps6xi4t4@wittgenstein/>) to look into it. Patches, he said, would be posted soon. 

#### PIDs: not dead yet

One other characteristic of the proposed `process_madvise()` API is that the caller must possess a pidfd to use it — unlike other system calls like [`kill()`](<http://man7.org/linux/man-pages/man2/kill.2.html>) or [`setpriority()`](<http://man7.org/linux/man-pages/man2/setpriority.2.html>), which take process IDs. That is not uncommon for new, process-oriented system calls; pidfds are the cool new kid on the block and they appear to be generally preferred. A pidfd unambiguously identifies a process; a process ID, instead, can vary from one namespace to the next and, should the target process exit, might be reused for an entirely unrelated process. For such reasons, pidfds have quickly become the favored way of identifying processes in new system calls; as Colascione [put it](<https://lwn.net/ml/linux-kernel/CAKOZuevwbQvrFWqy5GOm4RXuGszKLBvRs9i-KbAi3nPcHhwvSw@mail.gmail.com/>): ""All new APIs should use pidfds: they're better than numeric PIDs in every way"". 

It turns out, though, that not everybody agrees with that point of view. Process IDs are often already known, while a pidfd would have to be created; PIDs can also be specified by users on a command line, while pidfds cannot. Kirill Tkhai [was one of the dissenters](<https://lwn.net/ml/linux-kernel/3eec2097-75a3-1e1d-06d9-44ee5eaf1312@virtuozzo.com/>): 

Ordinary pid interfaces also should be available. There are a lot of cases, when they are more comfortable. Say, a calling of process_madvise() from tracer, when a tracee is stopped. In this moment the tracer knows everything about tracee state, and pidfd brackets pidfd_open() and close() around actual action look just stupid, and this is cpu time wasting. 

Thus, he [argued](<https://lwn.net/ml/linux-kernel/9d849087-3359-c4ab-fbec-859e8186c509@virtuozzo.com/>), every new process API should be able to handle both PIDs and pidfds. Kim took this request to heart, as can be seen in [a new version of the `process_madvise()` patch set](<https://lwn.net/ml/linux-kernel/20200116235953.163318-1-minchan@kernel.org/>) posted one week later. The API for this proposed system call now is: 

```
int process_madvise(int which, pid_t pid, void *addr, size_t length,
        			int advice, unsigned long flag);
```

The new `which` parameter would be either `P_PID` or `P_PIDFD` to tell the kernel how the `pid` argument should be interpreted. 

This change highlights a question that needs to be answered by the wider community. If there is a consensus that all new process-related system calls should support both ways of identifying processes, then this convention should really be applied universally and consistently to all new interfaces. Otherwise it perhaps should not be used even for `process_madvise()`. Creating a mix of APIs, some of which accept only one way while others support both, seems like the worst outcome. The Linux system-call API is inconsistent enough as it is now; future developers will not be grateful if new system calls make that situation worse.
