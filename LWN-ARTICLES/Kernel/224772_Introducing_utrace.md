---
title: Introducing utrace
url: https://lwn.net/Articles/224772/
date: "March 7, 2007"
category: Utrace; ptrace
author: 
---

The interface for tracing programs under Linux is the `ptrace()` system call. It is used primarily by debuggers, but there are other applications too; User-mode Linux can use `ptrace()`, for example. The interface gets the job done, but there are few system calls which endure more criticism. The list of `ptrace()` shortcomings is long, its interface is difficult for user-space developers to use and for kernel-space developers to maintain, it is inefficient, and it has been the source of more than one security problem over the years. Still, `ptrace()` endures; it is part of the user-space API and there is nothing better available. 

Soon there may be a better alternative, in the form of the "utrace" patch (by Roland McGrath) which is currently in the -mm tree. Utrace replaces `ptrace()` entirely, while maintaining the same interface to user space. As such, it is a useful cleanup of a difficult system call. The real value of utrace, however, is likely to be seen in new tracing interfaces in the future. 

The core utrace code does not interface with user space at all; instead, it is an in-kernel API which can be used to build kernel-based tracing mechanisms. These mechanisms are based around the concept of a "tracing engine," which is defined by the usual structure full of method pointers. This structure (`struct utrace_engine_ops`) has fourteen callbacks, each covering something which the traced process might do or have done to it. For example, one callback is: 

```
u32 (*report_syscall_entry)(struct utrace_attached_engine *engine,
    				struct task_struct *tsk,
    				struct pt_regs *regs);
```

Whenever the traced process invokes a system call, the tracing engine will (if it has asked for this event) receive a call to its `report_syscall_entry()` callback. The call happens at a "safe" time before the system call is executed; no locks are held, and the tracing process can safely access the traced process's state. The callback returns a bitmask specifying what happens next; the bitmask can change the tracing state, detach the engine, hide the event from other tracing engines, and more. 

A tracing engine is put into service with: 

```
struct utrace_attached_engine *
        utrace_attach(struct task_struct *target, int flags,
    	      	  const struct utrace_engine_ops *ops, 
    		  unsigned long data);
```

This call will attach the engine to the given `target` process. There can be more than one engine attached to any given process - a significant difference from `ptrace()`. A newly-attached engine does not actually do anything, one can think of it as being in an idling state. Putting the engine into gear requires setting one or more action flags with: 

```
int utrace_set_flags(struct task_struct *target,
    			 struct utrace_attached_engine *engine,
    			 unsigned long flags);
```

There is a special flag (`UTRACE_EVENT(QUIESCE)`) which puts the target process into a quiescent state. In general, operating on the task first requires setting this flag, then waiting for a callback (to the `report_quiesce()` engine method) that says the process is truly stopped. There is a whole other set of events which can be requested: forking, execing a new program, receiving a signal, process death, system call entry and exit, etc. Single-stepping through instructions and program blocks is also handled through the event mechanism. 

A signal can be forced into the target process with: 

```
int utrace_inject_signal(struct task_struct *target,
    			     struct utrace_attached_engine *engine,
    			     u32 action, siginfo_t *info,
    			     const struct k_sigaction *ka);
```

Signals injected in this manner are delivered to the target process immediately; they are not queued in the usual manner. 

There is more to the utrace API than is described in this brief overview, including an API for describing and working with CPU registers; see [the excellent documentation file](<https://lwn.net/Articles/224779/>) packaged with the patch for more details. Also included with the patch is a complete reimplementation of `ptrace()` built on top of utrace. 

Reimplementing `ptrace()` is only so interesting, however, even if the result is a big improvement. The real purpose behind utrace looks to be to inspire the creation of the next generation of user-space process tracing APIs, and more. Roland told your editor: 

The intent of the utrace API is not just to facilitate my writing the one great new userland API to replace ptrace. Its core purpose is to put writing a new user debugging facility more on par with writing a software device driver, a filesystem, or a network stack, so that many people can come up with ideas and experiment without doing brain surgery every time. It ties up the really nasty low-level implementation issues, and lets different unrelated facilities coexist without interfering with each other. 

In other words, while utrace should enable the eventual retirement of `ptrace()`, there is more coming than that. If and when utrace makes it into the mainline, look for it to inspire interesting developments in a number of areas.
