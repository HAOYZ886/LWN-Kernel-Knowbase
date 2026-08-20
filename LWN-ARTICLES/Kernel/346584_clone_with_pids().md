---
title: clone_with_pids()
url: https://lwn.net/Articles/346584/
date: "August 12, 2009"
category: Checkpointing; Containers
author: "By Jake Edge August 12, 2009"
---

By **Jake Edge**  
August 12, 2009

As part of the changes to support application [checkpoint and restart](<http://lwn.net/Articles/293575/>) in the kernel, Sukadev Bhattiprolu has [proposed](<http://lwn.net/Articles/345843/>) a new system call: `clone_with_pids()`. When a process that was checkpointed gets restarted, having the same process id (PID) as it had when the checkpoint was done is important to some kinds of applications. Normally, the kernel assigns an unused PID when a new task is started (via `clone()`), but, for checkpointed processes, that could lead to processes' PIDs changing during their lifetime, which could be an undesirable side effect. So, Bhattiprolu is looking for a way to avoid that by allowing `clone()` callers to specify the PID--or PIDs for processes in nested namespaces--of the child. 

The actual system call is fairly straightforward. It adds an additional `pid_set` parameter to `clone()`, to contain a list of process ids; `pid_set` has the obvious definition: 

```
struct pid_set {
    	   int num_pids;
    	   pid_t *pids;
        };
```

A pointer to a `pid_set` is passed as the last parameter to `clone_with_pids()`. Each of the PIDs is used to specify which PID should be assigned at each level of namespace nesting. The patch that actually implements `clone_with_pids()` (as opposed to the earlier patches in the patchset that prepare the way) [illustrates](<https://lwn.net/Articles/346627/>) this with an example (slightly edited for clarity): 

```
pid_t pids[] = { 0, 77, 99 };
    	struct pid_set pid_set;
    
    	pid_set.num_pids = sizeof(pids) / sizeof(int);
    	pid_set.pids = &pids;
    
    	clone_with_pids(flags, stack, NULL, NULL, NULL, &pid_set);
```

If a target-pid is 0, the kernel continues to assign a pid for the process in that namespace. In the above example, pids[0] is 0, meaning the kernel will assign next available pid to the process in init_pid_ns. But kernel will assign pid 77 in the child pid namespace 1 and pid 99 in pid namespace 2. If either 77 or 99 are taken, the system call fails with -EBUSY. 

The patchset assumes that being able to set PIDs is desirable, but Linus Torvalds was not particularly in favor of that approach when it was first [discussed](<http://lwn.net/Articles/324254/>) on linux-kernel back in March. His complaint was that there are far too many stateful attributes of processes to ever be able to handle checkpointing in the general case. His suggestion: ""just teach the damn program you're checkpointing that pids will change, and admit to everybody that people who want to be checkpointed need to do work"". 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Others disagreed--no surprise--but it is unclear that Torvalds has changed his mind. He was also concerned about the security implications of processes being able to request PID assignments: ""But it also sounds like a _wonderful_ attack vector against badly written user-land software that sends signals and has small races."" That particular concern should be alleviated by the requirement that a process have the `CAP_SYS_ADMIN` capability (essentially root privileges) in order to use `clone_with_pids()`. 

Requiring root to handle restarts, which in practice means that root must manage the checkpoint process as well, makes checkpoint/restart less useful, overall. But there are a whole host of problems to solve before allowing users to arbitrarily checkpoint and restore from their own, quite possibly maliciously crafted, checkpoint images. Even with root handling the process, there are a number of interesting applications.

There is an additional wrinkle that Bhattiprolu notes in the patch. Currently, all of the available `clone()` flags are allocated. That doesn't affect `clone_with_pids()` directly, as the flags it needs are already present, but, when adding a system call, it is good to look to the future. To that end, there are two proposed implementations of a `clone_extended()` system call, which could be added instead of `clone_with_pids()`, that would allow for more `clone()` flags, while still supporting the restart case. 

The first possibility is to turn the `flags` argument into a pointer to an array of flag entries, that would be treated like `signal()` sets, including operations to test, set, and clear flags _a la_ `sigsetops()`: 

```
typedef struct {
    	    unsigned long flags[CLONE_FLAGS_WORDS];
        } clone_flags_t;
    
        int clone_extended(clone_flags_t *flags, void *child_stack, int *unused,
    	    int *parent_tid, int *child_tid, struct pid_set *pid_set);
```

In the proposal, `CLONE_FLAGS_WORDS` would be set to 1 for 64-bit architectures, while on 32-bit architectures, it would be set to 2, thus doubling the number of available flags to 64. Should the number of clone flags needed grow, that could be expanded as required, though doing so in a backward-compatible manner is not really possible. 

Another option is to split the flags into two parameters, keeping the current `flags` parameter as it is, and adding a new `clone_info` parameter that contains new flags along with the `pid_set`: 

```
struct clone_info {
    	    int num_clone_high_words;
    	    int *flags_high;
    	    struct pid_set pid_set;
        }
    
        int clone_extended(int flags_low, void *child_stack, void *unused,
    	    int *parent_tid, int *child_tid, struct clone_info *clone_info);
```

There are pros and cons to each approach, as Bhattiprolu points out. The first requires a `copy_from_user()` for the flags in all cases (though 64-bit architectures might be able to avoid that for now), while the second requires the awkward splitting of the flags, but avoids the `copy_from_user()` for calls that don't use the new flags or `pid_set`s. 

It is hard to imagine that copying a bit of data from user space will measurably impact a system call that is creating a process, though, so some derivative of the first option would seem to be the better choice. It's also a bit hard to see the need for more than 64 `clone()` flags, but if that is truly desired, something with a path for compatibility is needed. 

There has been no objection to the implementation of `clone_with_pids()`, but there have been few comments overall. Pavel Machek [wondered](<https://lwn.net/Articles/346677/>) about the need for setting the PID of anything but the inner-most namespace, but Serge E. Hallyn [noted](<https://lwn.net/Articles/346678/>) that nested namespaces require that ability: ""we might be restarting an app using a nested pid namespace, in which case restart would specify pids for 2 (or more) of the innermost containers"".

Machek also thought there should be a documentation file that described the new system call, and Bhattiprolu agreed, but is waiting to see what kind of consensus on either `clone_with_pids()` or `clone_extended()` (and which of the two interfaces for the latter) would emerge. So far, no one has commented on that particular aspect. 

This is version 4 of the patchset, and the history shows that earlier comments have been addressed. It is still at the RFC stage, or, as Bhattiprolu puts it: ""Its mostly an exploratory patch seeking feedback on the interface"". That feedback has yet to emerge, however, and one might wonder whether Torvalds will still object to the whole approach. It would seem, though, that there are too many important applications for checkpoint and restart--including process migration and the ability to upgrade kernels underneath long-running processes--for some kind of solution not to make its way into the kernel eventually.
