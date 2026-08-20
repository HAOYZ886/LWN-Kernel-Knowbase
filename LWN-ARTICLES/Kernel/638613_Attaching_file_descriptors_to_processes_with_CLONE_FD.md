---
title: Attaching file descriptors to processes with CLONE_FD
url: https://lwn.net/Articles/638613/
date: "April 1, 2015"
category: "System calls-clone"
author: "By Jonathan Corbet April 1, 2015"
---

By **Jonathan Corbet**  
April 1, 2015

The classic Unix system design has many advantages, but friendliness to library code is not always one of them. The problem, at its core, is the use of global structures and mechanisms that cannot be manipulated independently by an application and any libraries it may use; the file descriptor table and signal handling vectors are a couple of examples. A recently posted patch set is trying to address a little piece of this problem, but may point the way toward a larger long-term change in how processes are managed in Linux. 

Consider a library that needs to create a child process with `fork()` and be notified when that process exits. The library cannot call `wait()` without blocking the entire calling process and, possibly, interfering with the reaping of any child processes created elsewhere in the application. Receiving asynchronous notifications with the `SIGCHLD` signal has a similar problem: there can only be one `SIGCHLD` handler, so catching it in the library clashes with the application's use of that signal. The need to avoid this kind of conflict with application code can greatly limit what can be done within a library. 

Josh Triplett's [`CLONE_FD`](<https://lwn.net/Articles/636646/>) patch set is an attempt to solve that particular problem. It adds a new flag, `CLONE_FD`, to a variant of the `clone()` system call (called `clone4()`); if that flag is present, a successful `clone4()` will return a file descriptor referring to the process it just created. There is little that can be done with that descriptor by default, but, if the `CLONE_AUTOREAP` flag is also set, the system's behavior changes somewhat. 

In particular, when a process created with `CLONE_AUTOREAP` exits, it will be cleaned up immediately rather than sitting in the "zombie" state until the parent process gets around to calling `wait()`. This behavior can be useful in its own right if the parent process is not concerned about when the child exits. If the parent does care about its child's exit, though, it can create a process file descriptor with `CLONE_FD`; that descriptor will become readable when the process exits. In particular, it will be possible to read a structure like: 

```
struct clonefd_info {
    	uint32_t code;   /* Signal code */
    	uint32_t status; /* Exit status or signal */
    	uint64_t utime;  /* User CPU time */
    	uint64_t stime;  /* System CPU time */
        };
```

With this mechanism in place, a library function can create a process and wait for it to complete without interfering with process management anywhere else in the program. It thus solves the problem of having to share the machinery for managing process exit information by removing that sharing and, essentially, turning a running process into a sort of "file" that can be tracked and managed exclusively by the code that knows about it. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

There was one tiny little problem in the way of implementing this solution, though: there is no room for additional flags for the `clone()` system call. So Josh had to create a new one, which he called `clone4()`. The intended interface, as presented by the C library, is: 

```
int clone4(uint64_t flags, size_t args_size, struct clone4_args *args,
                   int (*fn)(void *), void *arg);
```

The `flags` field holds the flags to the operation: the traditional `clone()` flags and the new ones described above. As with `clone()`, the child process will call `fn(arg)` after its creation. The `args` argument looks like this: 

```
struct clone4_args {
    	pid_t *ptid;
    	pid_t *ctid;
    	unsigned long stack_start;
    	unsigned long stack_size;
    	unsigned long tls;
    	int *clonefd;
    	unsigned clonefd_flags;
        };
```

Most of these fields match arguments passed to the `clone()` call; they have just been moved into a separate structure. The `clonefd` and `clonefd_flags` fields are new, though; the first is a location to store the created file descriptor when `CLONE_FD` is passed; the second can hold the `O_CLOEXEC` and `O_NONBLOCK` flags to be applied to that file descriptor. 

The size of the `clone4_args` structure must be passed separately in `args_size`. If fields must be added to this structure in the future, the passed size will allow the kernel to determine which version of the structure is in use and respond accordingly. 

For completeness, the actual system call (invoked from the C library wrapper) looks like this: 

```
int clone4(unsigned flags_high, unsigned flags_low,
                   unsigned long args_size,
                   struct clone4_args *args);
```

At this level, `clone4()` behaves like `fork()`, returning into both the parent and child processes (but with different return values). 

This functionality is useful enough, but the patch description also includes this intriguing note: 

The CLONE_FD file descriptor uniquely identifies a process on the system in a race-free way, by holding a reference to the task_struct. In the future, we may introduce APIs that support using process file descriptors instead of PIDs. 

Process IDs (PIDs) are subject to a narrow race condition: a process could exit and be replaced by another using the same PID. In a system where there are many processes running and there is a lot of fork activity, the probability of short-term PID reuse is significant, especially if, as is usually the case, the range of available PIDs has not been increased beyond the traditional 32,768. A file descriptor that is attached to a process at creation time is not subject to this race, though; it should thus be safer for other processes to operate on. 

Adding file-descriptor-oriented process-management system calls is a future exercise, though; for now, the `CLONE_FD` functionality solves the immediate problem. This patch has been through two rounds of review so far; some significant changes have been made, but the core functionality seems to be uncontroversial. One more review round seems likely to happen, though; after that, this feature could conceivably be added as early as the 4.2 development cycle.
