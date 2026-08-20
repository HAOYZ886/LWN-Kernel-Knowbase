---
title: "The beginning of a process-builder API"
url: https://lwn.net/Articles/1086330/
date: "August 4, 2026"
category: "System calls-clone"
author: "By Jonathan Corbet August 4, 2026"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
August 4, 2026

The recent [discussion on "spawn templates"](<https://lwn.net/Articles/1076018/>) raised questions about whether it was time to provide an alternative to the classic Unix `fork()`/`exec()` pattern for process creation. One idea that was raised there was to shift the template pattern into an interface that could be used to efficiently assemble new processes from bare cloth, without duplicating the parent process. Preferably, that interface would be able to implement [`posix_spawn()`](<https://man7.org/linux/man-pages/man3/posix_spawn.3.html>). Li Chen, the author of the spawn-template work, has now responded with [a patch series](<https://lwn.net/ml/all/cover.1784204592.git.me@linux.beauty>) (written with significant LLM assistance) showing what a process-builder API for Linux might look like. 

In the Unix model, a call to `fork()` (which ends up being a variant of [`clone()`](<https://man7.org/linux/man-pages/man2/clone.2.html>) on Linux systems) creates a copy of the calling process, which involves a fair amount of work. The child then typically modifies its environment in whatever ways are necessary — opening or closing files, for example — before making a call to [`execve()`](<https://man7.org/linux/man-pages/man2/execve.2.html>) to run a new program. That latter call ends up throwing away most of the work that was done to copy the parent process, which is not entirely efficient. In cases where the intent is to immediately run a different program, a better model might be to piece together the new process from the beginning, without involving (much of) the parent process's state. 

#### Process creation

In Chen's patch series, the way to do that is to start by creating an empty process with a call to the existing [`pidfd_open()`](<https://man7.org/linux/man-pages/man2/pidfd_open.2.html>) system call: 

```
new_process_fd = pidfd_open(0, PIDFD_EMPTY);
```

The new `PIDFD_EMPTY` flag requests the creation of a process shell that will have its details filled out later. The return value is a real pidfd, but most of the resources associated with a process are not yet present. There is no process ID, no task structure in the kernel, and no charge against the parent's process-count resource limit. Most kernel operations that act on a pidfd will refuse to do anything with this one at this stage. 

The next step is to put together the information that drives the construction of the new process; this work is centered around this structure: 

```
struct pidfd_spawn_run_args {
    	__u32 flags;
    	__u32 nr_actions;
    	__aligned_u64 path;
    	__aligned_u64 argv;
    	__aligned_u64 envp;
    	__aligned_u64 actions;
    	__u32 action_size;
    	__u32 reserved0;
    	__u64 reserved[2];
        };
```

The `path` field is a pointer to a string containing the path to the executable that the new process should run; as described below, it can be `NULL` in some cases. The `argv` and `envp` parameters point to the usual argument and environment arrays. The `flags` field must be zero, as must the reserved fields. Filling in those fields (the rest will be covered shortly) provides enough information to build and run the process with a call to the first of two new system calls: 

```
int pidfd_spawn_run(int pidfd, struct pidfd_spawn_run_args *args, int arg_size);
```

Here, `pidfd` is the pidfd for the under-construction process, `args` is a pointer to the above structure, and `arg_size` is the size of that structure. If all goes well, this call will create the full process and set it running with the indicated program; the return value will be the ID of the now fully fleshed-out process. The original pidfd remains valid and attached to this process. Should some sort of error happen, instead, the process shell will be left in a sort of dead state, and any further attempts to operate on it will fail. 

#### Configuration

Providing an executable image, arguments, and environment is normally just the beginning of configuring a new process; a typical application will want to set up the process's file descriptors and, perhaps, adjust many other things. That is what the `actions` field of the `pidfd_spawn_run_args` structure is for. It points to an array of this structure type: 

```
struct pidfd_spawn_action {
    	__u32 type;
    	__u32 flags;
    	__u32 fd;
    	__u32 newfd;
    	__u64 reserved[2];
        };
```

The `type` field describes an action that should be carried out before the process is instantiated and launched. The actions currently defined in this patch set (which are a small subset of what would eventually be needed) are: 

  * **`PIDFD_SPAWN_ACTION_DUP2`** : duplicates an existing file descriptor using [`dup2()`](<https://man7.org/linux/man-pages/man2/dup.2.html>). The existing file descriptor should be passed in the `fd` field, while the intended new descriptor goes in `newfd`. 
  * **`PIDFD_SPAWN_ACTION_CLOSE_RANGE`** : closes the range of file descriptors between `fd` and `newfd` (inclusive). 
  * **`PIDFD_SPAWN_ACTION_FCHDIR`** : changes the new process's working directory to the one identified by `fd`. 

`PIDFD_SPAWN_ACTION_CLOSE_RANGE` allows the `CLOSE_RANGE_CLOEXEC` and `CLOSE_RANGE_UNSHARE` flags supported by the [`close_range()`](<https://man7.org/linux/man-pages/man2/close_range.2.html>) system call; `flags` must be zero for the other two actions. The `nr_actions` field in the `pidfd_spawn_run_args` structure indicates how many actions are present, while `action_size` is the size of the `pidfd_spawn_action` structures. These actions will be carried out before the executable image is located, meaning that changing the working directory affects how a relative path to that image is resolved. 

If any of the actions fail at `pidfd_spawn_run()` time, the entire system call will fail and the new process will not be launched. 

The second new system call is an alternative configuration interface for some aspects of the new process: 

```
int pidfd_config(int pidfd, unsigned int cmd, char *ukey, char *value,
        		     int aux);
```

This system call is meant to be useful beyond the spawn functionality; in this patch set, though, it can only do one thing. With a `cmd` of `PIDFD_CONFIG_SET_STRING`, it can set a string-based parameter to the given `value`. The only such parameter in this set is `PIDFD_CONFIG_KEY_PATH`, which sets the path to the executable to run. This call must be made if the `path` pointer passed to `pidfd_spawn_run()` will be `NULL`. If a non-`NULL` path _is_ passed to that system call, it will override a path configured with `pidfd_config()`. 

#### What's missing

This series is meant to be a proof of the concept that would help the community decide whether the overall direction makes sense or not. So it does not implement much of what would be wanted in the final result. As mentioned above, the set of available actions is much smaller than it would eventually need to be. To be able to implement `posix_spawn()`, the kernel would have to support actions to define signal handling, control scheduler parameters, open files, and more. For the most part, these are just a matter of programming once the form of the desired interface is established. 

The other big gap is that `pidfd_spawn_run()` does not actually create a new process from scratch; instead, the implementation is based on `vfork()` internally. In theory, it should be possible to replace the implementation transparently, with the only change visible to user space being better performance, once the API is agreed upon. In practice, that may not be a small job. The kernel only ever creates two special-purpose processes (`init` and `kthreadd`) from scratch; the machinery to do that for the general case does not exist. That, too, is a matter of programming — perhaps a fair amount of it. 

First, though, the API must be agreed upon. The series was posted on July 16, but has only garnered a single review comment as of this writing. It will need to attract a lot more eyeballs before a fuller implementation can be considered. A number of people have been asking for this kind of process-creation interface for years; now would be a good time for them to take a look to see whether this proposal is close to what they have in mind.
