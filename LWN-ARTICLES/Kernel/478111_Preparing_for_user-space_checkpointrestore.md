---
title: "Preparing for user-space checkpoint/restore"
url: https://lwn.net/Articles/478111/
date: "January 31, 2012"
category: "Checkpointing; DMTCP; System calls-kcmp"
author: "By Jonathan Corbet January 31, 2012"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
January 31, 2012

The addition of a checkpoint/restore functionality to Linux has been an ongoing topic of discussion and development for some years now. After the [poor reception](<https://lwn.net/Articles/414264/>) given to the in-kernel C/R implementation at the end of 2010, that particular project seems to have faded into the background. Instead, most of the interest seems to be in solutions that operate mostly in user space. Depending on the approach taken, most or all the support needed to implement this functionality in user space already exists. But a complete solution is not yet there. 

#### CRIU

Cyrill Gorcunov has been working to fill in some of the gaps with [a preparatory patch set](<https://lwn.net/Articles/478117/>) for user-space checkpointing/restore with the "CRIU" tool set. There are a number of small additions to the kernel ABI to be found here: 

  * A new `children` entry in a thread's `/proc` directory provides a list of that thread's immediate children. This information allows a user-space checkpoint utility to find those child processes without needing to walk through the entire process tree. 

  * `/proc/_pid_ /stat` is extended to provide the bounds of the process's argument and environment arrays, along with the exit code. That allows this information to be reliably captured at checkpoint time. 

  * A number of new `prctl()` options allow the argument and environment arrays to restored in a way matching what was there at checkpoint time. The desired end result is that `ps` shows the same information about a process after a checkpoint/restore cycle as it did before. 

Perhaps the most significant new feature, though, is the addition of a new system call: 

```
long kcmp(pid_t pid1, pid_t pid2, int type, unsigned long idx1, unsigned long idx2);
```

Checkpoint/restore is meant to work as well on a tree of processes as on a single process. One challenge in the way of meeting that goal is that some of those processes may share resources - files, say, or, perhaps, a whole lot more. Replicating that sharing at restore time is relatively easy; the `clone()` system call provides a nice set of flags controlling the sharing of resources. The harder part is knowing, at checkpoint time, whether that sharing is taking place. 

One way for user space to determine whether, for example, two processes are sharing the same open file would be to query the kernel for the address of the associated `struct file` and see if they are the same in both processes. That kind of functionality sets off alarms among those concerned about security, though; learning where data structures live in kernel space is often an important precondition to an attack. There was talk for a while of "obfuscating" the pointers - through an exclusive-OR with a random value, for example - but the risk was still seen as being too high. So the compromise is `kcmp()`, which simply answers the question of whether resources found in two processes are the same or not. 

`kcmp()` takes two process ID parameters, indicating the processes of interest; both processes must be in the same PID namespace as the calling process. The `type` parameter tells the kernel the specific item that is being compared: 

  * `KCMP_FILE`: determines whether a file descriptor `idx1` in the first process is the same as another descriptor (`idx2`) in the second process. 

  * `KCMP_FILES`: compares the file descriptor arrays to see whether the processes share all files. 

  * `KCMP_FS`: compares `fs_struct` structures (which hold the current umask, working directory, namespace root, etc.). 

  * `KCMP_IO`: compares the I/O context, used mainly for block I/O scheduling. 

  * `KCMP_SIGHAND`: compares the two process's signal handler arrays. 

  * `KCMP_SYSVSEM`: compares the list of undo operations associated with SYSV semaphores. 

  * `KCMP_VM`: compares each process's address space. 

The return value from `kcmp()` is zero if the two items are equal, one if the first item is "less" than the second, or two if the first is "greater" than the second. The ordered comparison may seem a little strange, especially when one looks at the implementation and sees that the pointers are obfuscated before comparison within the kernel. The result is, thus, an ordering that (by design) does not match the ordering of the relevant data structures in kernel space. It turns out that even a reshuffled (but consistent) "ordering" is useful for optimizing comparisons in user space when large numbers of open files are present. 

This patch set has been through a few cycles of review and seems to have addressed most of the concerns raised by reviewers. It may just find its way in through the next merge window. Meanwhile, people who want to see how the user-space side works can find the relevant code at [criu.org](<http://criu.org>). 

#### DMTCP

CRIU is not the only user-space checkpoint/restore implementation out there; the [DMTCP (Distributed MultiThreaded CheckPointing)](<http://dmtcp.sourceforge.net/>) project has been busy since about 2.6.9. DMTCP differs somewhat from CRIU, though; in particular, it is able to checkpoint groups of processes connected by sockets - even across different machines - and it requires no changes to the kernel at all. These features come with a couple of limitations, though. 

Checkpoint/restore with DMTCP requires that the target process(es) be started with a special script; it is not possible to checkpoint arbitrary processes on the system. That script uses the LD_PRELOAD mechanism to place wrappers around a number of libc and (especially) system call implementations. As a result, DMTCP has no need to ask the kernel whether two processes are sharing a specific resource; it has been watching the relevant system calls and knows how the processes were created. The disadvantage to this approach - beyond having to run checkpointable process in a special environment - is that, as can be seen in [the table of supported applications](<http://dmtcp.sourceforge.net/supportedApps.html>), not all programs can be checkpointed. 

The recent [1.2.4 release](<http://sourceforge.net/news/?group_id=194616>) improves support, though, to the point that everything a wide range of users care about should be checkpointable. The system has been integrated with [Open MPI](<http://www.open-mpi.org/>) and is able to respond to MPI-generated checkpoint and restore requests. DMTCP is available with the openSUSE, Debian Testing, and Ubuntu distributions. DMTCP may offer something good enough today for many users, who may not need to wait for one of the other projects to be ready sometime in the future.
