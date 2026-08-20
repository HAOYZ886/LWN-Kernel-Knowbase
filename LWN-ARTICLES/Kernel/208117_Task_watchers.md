---
title: Task watchers
url: https://lwn.net/Articles/208117/
date: "November 7, 2006"
category: Notifiers; Task watchers
author: 
---

One of the more complicated core kernel functions is `copy_process()`, in `kernel/fork.c`. This routine is the heart of the `fork()` and `clone()` system calls; it must create a coherent copy of a running process, bearing in mind the various clone flags which are present. There are sixteen different `goto` labels for error exits. This is clearly a place where a lot of things can go wrong. 

It is also an operation of interest to many other kernel subsystems. A look at `copy_process()` reveals hooks for task delay accounting, auditing, the process fork connector, SYSV semaphore undo information management, NUMA memory policy enforcement, cpuset maintenance, keyring management, and more. Many of these subsystems want to know about other events in the process lifecycle as well, with the result that hooks are placed all over the process code. It might just be nice to have a cleaner solution to the problem of learning about process-related events. 

That cleaner solution would appear to be present in the form of Matt Helsley's [task watchers patch set](<http://lwn.net/Articles/207873/>), currently in its second major iteration. This patch takes an interesting approach to providing what is essentially just another notifier interface in order to minimize overhead in a performance-critical part of the kernel. 

In this patch, a "task watcher" is a function which is notified whenever an interesting process event takes place. Watchers have this prototype: 

```
int my_watcher(unsigned long info, struct task_struct *tsk);
```

When the watcher function is called, `info` will have additional information for the specific event, and `tsk` points to the process generating the event. Arranging for a task watcher to be called is a simple matter of adding a declaration like the following: 

```
task_watcher_func(event, function);
```

Where `event` is the event of interest, and `function` is the task watcher function to be called in response to that event. The possible events are: 

  * `init`: a process is first created; `info` is the set of flags passed to `clone()`. 

  * `clone`: a process forks; `info` is the set of `clone()` flags. Note that this watcher appears to be called with the child process; it differs from `init` in that it is called toward the end of `copy_process()`, when creation of the new process is complete. 

  * `exec`: a process executes a new program; `info` is zero. 

  * `uid`: a process changes its real or effective UID; `info` is zero. 

  * `gid`: a process changes its real or effective GID; `info` is zero. 

  * `exit`: a process dies; `info` is the exit code. 

  * `free`: a process's task structure is being freed; `info` is the exit code. 

The `task_watcher_func()` macro creates a pointer to the watcher function in a special ELF section. There is a separate section for each watched-for event; when such an event is signaled, the watcher code simply iterates through each function found in the relevant executable section. There are a couple of implications resulting from this mechanism: task watchers exist for the life of the system (they cannot be registered and unregistered), and they cannot be located in loadable modules (though this restriction will eventually go away). 

One might well wonder why things were done this way, rather than using a simple notifier list. Your editor wondered, and asked Mr. Helsley about it. The problem is that process creation is a performance-critical part of the kernel, and any change which increases process fork time tends to get a lot of scrutiny. Fork times are measured by a number of benchmarks; quick process creation is also important in fork-heavy loads. Since kernel compilation can require a lot of forks, there is an especially strong incentive to keep it fast. 

If a notifier list is used with watchers, some sort of locking is required to keep that list from being corrupted when watchers come and go. The separate ELF sections, instead, are read-only structures created at kernel build time. So they impose less overhead on the process lifecycle and, thus, are less likely to bother kernel developers who, perhaps, are not really interested in the watcher functionality.
