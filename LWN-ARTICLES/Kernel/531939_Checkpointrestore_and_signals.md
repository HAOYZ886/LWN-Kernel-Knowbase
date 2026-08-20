---
title: Checkpoint/restore and signals
url: https://lwn.net/Articles/531939/
date: "January 9, 2013"
category: "Checkpointing; System calls-signalfd"
author: "By Michael Kerrisk January 9, 2013"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Michael Kerrisk**  
January 9, 2013

Checkpoint/restore is a mechanism that permits taking a snapshot of the state of an application (which may consist of multiple processes) and then later restoring the application to a running state. One use of checkpoint/restore is for live migration, which allows a running application to be moved between host systems without loss of service. Another use is incremental snapshotting, whereby periodic snapshots are made of a long-running application so that it can be restarted from a recent snapshot in the event of a system outage, thus avoiding the loss of days of calculation. There are also many other uses for the feature. 

Checkpoint/restore has a long history, which [we covered](<https://lwn.net/Articles/525675/>) in November. The initial approach, starting in 2005, was to provide a kernel-space implementation. However, the patches implementing this approach were ultimately rejected as being too complex, invasive, and difficult to maintain. This led to an alternate approach: checkpoint/restore in user space (CRIU), an implementation that performs most of the work in user space, with some support from the kernel. The benefit of the CRIU approach is that, by comparison with a kernel-space implementation, it requires fewer and less invasive changes in the kernel code. 

To correctly handle the widest possible range of applications, CRIU needs to be able to checkpoint and restore as much of a process's state as possible. This is a large task, since there are very many pieces of process state that need to be handled, including process ID, parent process ID, credentials, current working directory, resource limits, timers, open file descriptors, and so on. Furthermore, some resources may be shared across multiple processes (for example, multiple processes may hold open file descriptors referring to the same open file), so that successfully restoring application state also requires reproducing shared aspects of process state. 

For each piece of process state, CRIU requires two pieces of support from the kernel: a mechanism for retrieving the state (used during checkpoint) and a mechanism to set the state (used during restore). In some cases, the kernel provides most or all of the necessary support. In other cases, however, the kernel does not provide a mechanism to retrieve the (complete) value of the state during a checkpoint or does not provide a mechanism to set the state during restore. Thus, one of the ongoing pieces of work for the implementation of CRIU is to add support to the kernel for these missing pieces. 

Andrey Vagin's recent patches to the `signalfd()` system call are an example of this ongoing work and illustrate the complexity of the task of saving and restoring process state. Before looking at these patches closely, we need to consider the general problem that CRIU is trying to solve with respect to signals, and consider some of the details that make the solution complicated. 

#### The problem and its complexities

The overall problem that the CRIU developers want to solve is checkpointing and restoring a process's set of pending signals--the set of signals that have been queued for delivery to the process but not yet delivered. The idea is that when a process is checkpointed, all of the process's pending signals should be fetched and saved, and when the process is restored, all of the signals should be requeued to the process. As things stand, the kernel does not quite provide sufficient support for CRIU to perform either of these tasks. 

At first glance, it might seem that the task is as simple as fetching the list of pending signal numbers during a checkpoint and then requeueing those signals during the restore. However, there's rather more to the story than that. First, each signal has an associated `siginfo` structure that provides additional information about the signal. That information is available when a process receives a signal. If a signal handler is installed using `sigaction()` with the `SA_SIGINFO` flag, then the additional information is available as the second argument of the signal handler, which is prototyped as: 

```
void handler(int sig, siginfo_t *siginfo, void *ucontext);
```

The `siginfo` structure contains a number of fields. One of these, `si_code`, provides further information about the origin of the signal. A positive number in this field indicates that the signal was generated by the kernel; a negative number indicates that the signal was generated by user space (typically by a library function such as `sigqueue()`). For example, if the signal was generated because of the expiration of a POSIX timer, then `si_code` will be set to the value `SI_TIMER`. On the other hand, if a `SIGCHLD` signal was delivered because a child process changed state, then `si_code` is set to one of a range of values indicating that the process terminated, was killed by a signal, was stopped, and so on. 

Other `siginfo` fields provide further information about the signal. For example, if the signal was sent using the `kill()` system call, then the `si_pid` field contains the PID and the `si_uid` field contains the real user ID of the sending process. Various other fields in the `siginfo` structure provide information about specific signals. 

There are other factors that make checkpoint/restore of signals complicated. One of these is that multiple instances of the so-called real-time signals can be queued. This means that the CRIU mechanism must ensure that all of the queued signals are gathered up during a checkpoint. 

One final detail about signals must also be handled by CRIU. Signals can be queued either to a specific thread or to a process as a whole (meaning that the signal can be delivered to any of the threads in the process). CRIU needs a mechanism to distinguish these two queues during a checkpoint operation, so that it can later restore them. 

#### Limitations of the existing system call API

At first glance it might seem that the [`signalfd()`](<http://man7.org/linux/man-pages/man2/signalfd.2.html>) system call could solve the problem of gathering all pending signals during a CRIU checkpoint: 

```
int signalfd(int fd, const sigset_t *mask, int flags);
```

This system call creates a file descriptor from which signals can be "read." Reads from the file descriptor return `signalfd_siginfo` structures containing much of the same information that is passed in the `siginfo` argument of a signal handler. 

However, it turns out that using `signalfd()` to read all pending signals in preparation for a checkpoint has a couple of limitations. The first of these is that `signalfd()` is unaware of the distinction between thread-specific and process-wide signals: it simply returns all pending signals, intermingling those that are process-wide with those that are directed to the calling thread. Thus, `signalfd()` loses information that is required for a CRIU restore operation. 

A second limitation is less obvious but just as important. As we noted above, the `siginfo` structure contains many fields. However, only some of those fields are filled in for each signal. (Similar statements hold true of the `signalfd_siginfo` structure used by `signalfd()`.) To simplify the task of deciding which fields need to be copied to user space when a kernel-generated signal is delivered (or read via a `signalfd()` file descriptor), the kernel encodes a value in the two most significant bytes of the `si_code` field. The kernel then elsewhere uses a `switch` statement based on this value to select the code that copies values from appropriate fields in the kernel-internal `siginfo` structure to the user-space `siginfo` structure. For example, for signals generated by POSIX timers, the kernel encodes the value `__SI_TIMER` in the high bytes of `si_code`, which indicates that various timer-related fields must be copied to the user-space `siginfo` structure. 

Encoding a value in the high bytes of the kernel-internal `siginfo.si_code` field serves the kernel's requirements when it comes to implementing signal handlers and `signalfd()`. However, one piece of information is _not_ copied to user space. For kernel-generated signals (i.e., those signals with a positive `si_code` value), the value encoded in the high bytes of the `si_code` field is discarded before that field is copied to user space, and it is not possible for CRIU to unambiguously reconstruct the discarded value based only on the signal number and the remaining bits that are passed in the `si_code` field. This means that CRIU can't determine which other fields in the `siginfo` structure are valid; in other words, information that is essential to perform a restore of pending signals has been lost. 

A related limitation in the system call API affects CRIU restore. The obvious candidates for restoring pending signals are two low-level system calls, `rt_sigqueueinfo()` and `rt_tgsigqueueinfo()`, which queue signals for a process and a thread, respectively. These system calls are normally rarely used outside of the C library (where, for example, they are used to implement the `sigqueue()` and `pthread_sigqueue()` library functions). Aside from the thread-versus-process difference, these two system calls are quite similar. For example, `rt_sigqueueinfo()` has the following prototype: 

```
int rt_sigqueueinfo(pid_t tgid, int sig, siginfo_t *siginfo);
```

The system call sends the signal `sig`, whose attributes are provided in `siginfo`, to the process with the ID `tgid`. This seems perfect, except that the kernel imposes one limitation: `siginfo.si_code` must be less than 0. (This restriction exists to prevent a process from spoofing as the kernel when sending signals to other processes.) This means that even if we could use `signalfd()` to retrieve the two most significant bytes of `si_code`, we could not use `rt_sigqueueinfo()` to restore those bytes during a CRIU restore. 

#### Progress towards a solution

Andrey's [first attempt](<https://lwn.net/Articles/530773/>) to add support for checkpoint/restore of pending signals took the form of an extension that added three new flags to the `signalfd()` system call. The first of these flags, `SFD_RAW`, changed the behavior of subsequent reads from the signalfd file descriptor: instead of returning a `signalfd_siginfo` structure, reads returned a "raw" `siginfo` structure that contains some information not returned via `signalfd_siginfo` and whose `si_code` field includes the two most significant bytes. The other flags, `SFD_PRIVATE` and `SFD_GROUP`, controlled whether reads should return signals from the per-thread queue or the process-wide queue. 

One other piece of the patch set relaxed the restrictions in `rt_sigqueueinfo()` and `rt_tgsigqueueinfo()` so that a positive value can be specified in `si_code`, so long as the caller is sending a signal to itself. (It seems safe to allow a process to spoof as the kernel when sending signals to itself.) 

A [discussion](<https://lwn.net/Articles/532105/>) on the design of the interface ensued between Andrey and Oleg Nesterov. Andrey noted that, for backward compatibility reasons, the `signalfd_siginfo` structure could not be fixed to supply the information required by CRIU, so a new message format really was required. Oleg noted that nondestructive reads that employed a positional interface (i.e., the ability to read message N from the queue) would probably be preferable. 

In response to Oleg's feedback, Andrey has now produced a [second version](<https://lwn.net/Articles/531942/>) of his patches with a revised API. The `SFD_RAW` flag and the use of a "raw" `siginfo` structure remain, as do the changes to `rt_sigqueueinfo()` and `rt_tgsigqueueinfo()`. However, the new patch set provides a rather different interface for reading signals, via the `pread()` system call: 

```
ssize_t pread(int fd, void *buf, size_t count, off_t offset);
```

In normal use, `pread()` reads `count` bytes from the file referred to by the descriptor `fd`, starting at byte `offset` in the file. Andrey's patch repurposes the interface somewhat in order to read from signalfd file descriptors: `offset` is used to both select which queue to read from and to specify an ordinal position in that queue. The caller calculates the `offset` argument using the formula 

```
queue + pos
```

`queue` is either `SFD_SHARED_QUEUE_OFFSET` to read from the process-wide signal queue, or `SFD_PER_THREAD_QUEUE_OFFSET` to read from the per-thread signal queue. `pos` specifies an ordinal position (not a byte offset) in the queue; the first signal in the queue is at position 0. For example, the following call reads the fourth signal in the process-wide signal queue: 

```
n = pread(fs, &buf, sizeof(buf), SFD_SHARED_QUEUE_OFFSET + 3);
```

If there is no signal at position `pos` (i.e., an attempt was made to read past the end of the queue), `pread()` returns zero. 

Using `pread()` to read signals from a signalfd file descriptor is nondestructive: the signal remains in the queue to be read again if desired. 

Andrey's second round of patches has so far received little comment. Although Oleg proposed the revised API, [he is unsure](<https://lwn.net/Articles/532138/>) whether it will pass muster with Linus: 

I think we should cc Linus. 

This patch adds the hack and it makes signalfd even more strange. 

Yes, this hack was suggested by me because I can't suggest something better. But if Linus dislikes this user-visible API it would be better to get his nack right now. 

To date, however, a version of the patches that copies Linus does not seem to have gone out. In the meantime, Andrey's work serves as a good example of the complexities involved in getting CRIU to successfully handle checkpoint and restore of each piece of process state. And one way or another, checkpoint/restore of pending signals seems like a useful enough feature that it will make it into the kernel in some form, though possibly with a better API.
