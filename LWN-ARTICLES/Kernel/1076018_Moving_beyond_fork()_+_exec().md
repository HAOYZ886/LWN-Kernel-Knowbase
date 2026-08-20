---
title: Moving beyond fork() + exec()
url: https://lwn.net/Articles/1076018/
date: "June 5, 2026"
category: "System calls-clone; System calls-execve"
author: "By Jonathan Corbet June 5, 2026"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
June 5, 2026

Since the earliest days of Unix, two of the core process-oriented system calls have been `fork()`, which creates a child process as a copy of the parent, and `exec()`, which runs a new program in the place of the current one. In Linux kernels, those system calls are better known as [`clone()`](<https://man7.org/linux/man-pages/man2/clone.2.html>) and [`execve()`](<https://man7.org/linux/man-pages/man2/execve.2.html>), but the core functionality remains the same. While there is elegance to this process-creation model, there are shortcomings as well. A recent [proposal](<https://lwn.net/ml/all/20260528095235.2491226-1-me@linux.beauty>) from Li Chen to add "spawn templates" to the kernel will not be accepted in its current form, but it may point the way toward a new process-creation primitive in the future. 

`fork()` is a relatively expensive system call; it must copy the entire process state (including memory) for the child process. Many optimizations have been made over the years, but a fork is still a fundamentally costly operation. To make things worse, a `fork()` call is often immediately followed by an `exec()`, which will discard all of that memory that was so carefully copied for the child. Attempts (such as [`vfork()`](<https://man7.org/linux/man-pages/man2/vfork.2.html>)) have been made over the years to optimize for this case, but the pattern still is more expensive than it could be. 

#### Spawn templates

Chen's patch set takes an interesting approach to optimize the `fork()` and `exec()` pattern. It is focused on applications that repeatedly launch processes running the same executable; imagine, for example, a program that must run Git repeatedly to obtain information about the contents of a repository. In such cases, the program could establish a template to accelerate those invocations, spreading the setup cost across multiple operations. This template would be created with the `spawn_template_create()` system call: 

```
struct spawn_template_create_args {
    	__aligned_u64 flags;
    	__s32 execfd;
    	__u32 exec_flags;
    	__aligned_u64 filename;
    	/* Some fields elided */
        };
    
        int spawn_template_create(struct spawn_template_create_args *args, size_t args_size);
```

This call will return a file descriptor representing a template for the executable file, which can be specified as either a file descriptor (`execfd`) or an absolute path (`filename`), but not both. To create the template, the kernel will open the indicated file and cache a bunch of information that will allow a process to run that file more quickly in the future. 

The application in question may run a given executable many times, but each invocation is different in a number of ways. The details of a specific invocation must be placed into an instance of this structure: 

```
struct spawn_template_spawn_args {
    	__aligned_u64 flags;
    	__aligned_u64 pidfd;
    	__aligned_u64 argv;
    	__aligned_u64 envp;
    	__aligned_u64 actions;
    	__aligned_u64 actions_len;
    	__aligned_u64 reserved[4];
        };
```

The `argv` field is a pointer to the argument list to be passed to the program, while `envp` points to its environment. Changes to file descriptors and signal handling, instead, are passed through `actions`, which is a pointer to an array of: 

```
struct spawn_template_action {
    	__u32 type;
    	__u32 flags;
    	__s32 fd;
    	__s32 newfd;
    	__aligned_u64 arg;
        };
```

If, for example, file descriptor four should be closed in the child, the associated `spawn_template_action` structure would have `type` set to `SPAWN_TEMPLATE_ACTION_CLOSE` and `fd` set to four. Other actions exist for duplicating file descriptors, opening files, changing the working directory, and changing signal handling. 

Once the `spawn_template_spawn_args` structure has been filled in, the new process can be run with: 

```
int spawn_template_spawn(int template_fd,
        			     struct spawn_template_spawn_args *args, int args_size);
```

Internally, this system call follows something close to the normal `fork()`/`exec()` path. Chen is careful to point out that all of the normal checks applied when executing a new file remain in place. But the cached information in the template makes the whole process faster than it was before. How much faster? Benchmark results provided in the cover letter show an improvement of about 2%, which may not seem like a lot, but it may make a difference for applications that fit the expected pattern. 

#### Toward `posix_spawn()`

The most detailed review of this work was [posted by Mateusz Guzik](<https://lwn.net/ml/all/vealb52tv5suireenkke4lul2l3wbnaul2rp3ea545ly5wa5ty@yk3aksvp7skt>), who said: ""This problem is dear to my heart and I have been pondering it on and off for some time now. The entire fork + exec idiom is terrible and needs to be retired"". He pointed out that the focus of the patch set was a bit strange in that it left the `fork()` part of the problem untouched. That is where most of the cost lies, he said, so optimization efforts should seek to remove it from the picture. Rather than copying the current process, ""creating a pristine process is the way to go"". 

Christian Brauner [was favorable](<https://lwn.net/ml/all/20260528-madig-fachrichtung-fehlinformation-61117ba640da@brauner>) toward the goal, saying: ""The idea of having a builder api for exec isn't all that crazy"". His suggestion, though, was that a new API should be built on top of the existing [pidfd](<https://lwn.net/Articles/794707/>) abstraction. Without getting into any degree of detail, he said that the right approach would be to create an option to [`pidfd_open()`](<https://man7.org/linux/man-pages/man2/pidfd_open.2.html>) to create an empty process. A series of calls to a new `pidfd_config()` system call would then configure this new process as desired, setting up its environment, image to execute, and more. `pidfd_config()` would thus be analogous to [`fsconfig()`](<https://man7.org/linux/man-pages/man2/fsconfig.2.html>). 

An important objective for a new interface, Brauner said, would be the ability to support an implementation of [`posix_spawn()`](<https://man7.org/linux/man-pages/man3/posix_spawn.3.html>) in user space. `posix_spawn()` is well suited as a replacement for the `fork()`/`exec()` pattern; developers would likely welcome a native implementation that isn't (unlike the current implementation) hiding `fork()` and `exec()` under the covers. Chen [agreed](<https://lwn.net/ml/all/19e883b2f84.6134d346323880.1325813164715871999@linux.beauty>) that the API as broadly sketched out by Brauner seemed better, and said that future work would be in that direction. So there will be no spawn templates in the Linux kernel but, if Chen's future work comes to fruition, Linux may finally gain a proper `posix_spawn()` implementation instead.
