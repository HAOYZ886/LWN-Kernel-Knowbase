---
title: "Toward race-free process signaling"
url: https://lwn.net/Articles/773459/
date: "December 6, 2018"
category: "System calls-pidfd send signal; pidfd"
author: "December 6, 2018 This article was contributed by Marta Rybczyńska"
---

December 6, 2018

This article was contributed by Marta Rybczyńska

Signals have existed in Unix systems for years, despite the general consensus that they are an example of a [bad design](<https://lwn.net/Articles/414618/>). Extensions and new ways of using signals pop up from time to time, fixing the issues that have been found. A notable addition was the introduction of [`signalfd()`](<http://man7.org/linux/man-pages/man2/signalfd.2.html>) nearly 10 years ago. Recently, the kernel developers have discussed how to avoid race conditions related to process-ID (PID) recycling, which occurs when a process terminates and another one is assigned the same PID. A process that fails to notice that its target has exited may try to send a signal to the wrong recipient, with potentially grave consequences. A patch set from Christian Brauner is trying to solve the issue by adding signaling via file descriptors.

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

PIDs increase for each new process up to the maximum value, and then go back to the beginning. For the maximum value, most distributions use the conservative value of 32768 to avoid breaking legacy systems. However, users can consult and change the maximum value in `/proc/sys/kernel/pid_max`. Signal-related APIs identify processes by PID. The disadvantage of this method is that, in the lifetime of a system, the same PID is reused as processes are created and terminated. What happens if a process has finished and another one has taken its PID? The PID value stays valid. Other processes, unaware of the situation, may try to send signals to the wrong process. This may have consequences as serious as terminating the wrong service. This race condition requires the PID space to wrap between the creation of the two processes, which is not uncommon.

#### `/proc/_pid_ /kill` proposal

The recent discussion started when Daniel Colascione [proposed adding a file called `kill`](<https://lwn.net/ml/linux-kernel/20181029221037.87724-1-dancol@google.com/>) to each process's `/proc` directory. Writing the numerical value of the desired signal to that file would send that signal to the selected process. The race was solved — or attempted to be solved — by holding the `/proc` directory open, thus preventing the PID from being reused at the wrong time. The [discussion showed](<https://lwn.net/ml/linux-kernel/20181030104037.73t5uz3piywxwmye@gmail.com/>) that other developers were considering the same problem in parallel.

While the problem was well understood, a debate started about the implementation. One part of the discussion concerned whether opening the `/proc` directory is enough to prevent PID reuse, in which case the patch would not have not been necessary in the first place. A small test case was developed and showed that [the answer is no](<https://lwn.net/ml/linux-kernel/CAKOZueuBdY_b=CVk29pzn0NAt9sJtzr6yqx9Y00WCppCm1JFWQ@mail.gmail.com/>). A modification in the kernel, like the proposed patch, is needed to solve the issue.

A heated debate followed on a proper API to deliver the signal: should it be a write to the file or another system call? Jann Horn [suggested an `ioctl()`](<https://lwn.net/ml/linux-kernel/CAG48ez01OWYVqEo+Qf65scG7a-48tDM4Gh3BOUKY+-b9g+9itw@mail.gmail.com/>), [followed](<https://lwn.net/ml/linux-kernel/20181031181717.GD2180@cisco/>) by Tycho Andersen, who agreed that this would simplify the permission checks. Colascione replied by supporting the choice of using `write()` and stated that it is unsafe to [call `ioctl()` on an unknown descriptor](<https://lwn.net/ml/linux-kernel/CAKOZuetdSK8jhD1snja8p4AQTNPrLOugQrVn642RCH8S4QBGFg@mail.gmail.com/>). In case of a mistake, we do not know what the effect of a random operation will be (this is, however, similar when writing to an unknown file). His other option was adding a new system call.

Another part of the debate was started by Aleksa Sarai who [added namespaces](<https://lwn.net/ml/linux-kernel/20181030050012.u43lcvydy6nom3ul@yavin/>) to the mix: there is a risk of processes sending signals between PID namespaces in situations when they normally do not have that ability. He suggested that only processes from the same PID namespace should be able to send such signals.

In the same thread, Brauner mentioned that he is working on a [similar solution](<https://lwn.net/ml/linux-kernel/20181030103910.mnzot3zcoh6j7did@gmail.com/>) and proposed postponing the patch review to after the discussion at Linux Plumbers Conference that was then two weeks away. Colascione and other developers were interested to know more. Colascione also [added some context to the discussion](<https://lwn.net/ml/linux-kernel/CAKOZuesuj6HyX-Y6J7wJ73PB_EuoKc2OB68nYzWY-08i+M+z-g@mail.gmail.com/>) by noting that there had been previous, but unsuccessful, attempts to solve this problem. There are two options to fix the issue and keep the interface race-free: either keep a PID reserved when the handle is open, or keep a reference to the `struct pid` instead of the PID value.

#### Signaling by `/proc/_pid_`

After LPC, Brauner [submitted a new patch set](<https://lwn.net/ml/linux-kernel/20181119103241.5229-1-christian@brauner.io/>). It proposes to solve the signal delivery issue by using file descriptors to identify processes; these descriptors would be obtained by opening a process's `/proc` directory. The solution Brauner proposed is to store a handle to the process's `struct pid` in the inode associated with that descriptor; this gives a stable handle that does not have the disadvantages of the simple PID number. It turned out that the patch could be simplified; Eric W. Biederman [explained](<https://lwn.net/ml/linux-kernel/87o9ak28nl.fsf@xmission.com/>) that the handle is already present in the inode reference and `proc_pid()` is enough to get the handle from a file descriptor.

While the first part of the patch deals with getting the handle, the second part of the patch set implements sending the signal itself; it is done using a new system call named `procfd_signal()`. This system call operates on a file descriptor of a process; the previous discussions convinced Brauner that this is a solution preferred over an `ioctl()`. The new system call has the following prototype:

```
long procfd_signal(int fd, int sig, siginfo_t *info, int flags);
```

It sends the signal `sig` to the process identified by the file descriptor `fd`. The optional `info` argument is a pointer to `siginfo_t` provided by the caller (used when sending realtime signals), and `flags` is reserved for future use and should be zero. On success, the system call returns zero; in case of an error it returns -1 and `errno` is set to the detailed error code: `EBADF` if the given file descriptor is not valid, `EINVAL` if the signal value is invalid or the file descriptor does not refer to a process, `EPERM` if the caller does not have sufficient permissions to send a signal to the target, and `ESRCH` if the target process does not exist.

The submission caused discussions of both the implementation and the use of signaling via file descriptors. While there has been no direct opposition, the developers noted a number of issues that should be taken into account. Sarai [started a discussion about sending signals to other namespaces](<https://lwn.net/ml/linux-kernel/20181119202857.k5zw742xjfrw677j@yavin/>). As a result, a check has been added so that sending signals is possible only to processes in child PID namespaces. This avoids problems when file descriptors leak between namespaces, for example when the root file system is bind-mounted into a container. Adding the possibility to send signals to ancestors [can be always added in the future](<https://lwn.net/ml/linux-kernel/20181121213946.GA10795@mail.hallyn.com/>).

The debate on which system call to use restarted with Andersen [again preferring an `ioctl()` interface](<https://lwn.net/ml/linux-kernel/20181119223954.GA4992@cisco/>). Colascione and Brauner argued instead for a new system call. This kind of debate happens quite often in the kernel community. Some developers prefer adding a new `ioctl()`, because they think that adding a system call is too complex. On the other hand, `ioctl()` is considered a worse API. Andy Lutomirski added a twist to the debate and [proposed a better version of the `ioctl()` system call](<https://lwn.net/ml/linux-kernel/CALCETrWyuQvTtksN1J1XbCFPka_rLOaFqa5W==EvGQvoaf9f3Q@mail.gmail.com/>). The discussion finished without a clear conclusion.

An example of how to use the mechanism has been posted in the cover letter. It is simple: the programs opens the right `/proc/_pid_` directory and then sends the signal with all parameters.

The [second version](<https://lwn.net/ml/linux-kernel/20181120105124.14733-1-christian@brauner.io/>) of the submission included both 32 and 64-bit versions of the system call with two different entry points. Lutomirski [objected](<https://lwn.net/ml/linux-kernel/CALCETrWnQNMQcCmFZrftVVYgAMW6DT3gyxvVb_v9_enUCUkHiw@mail.gmail.com/>), explaining that this design should be avoided for new system calls. The two versions are necessary due to the differing definition of `struct siginfo_t`. An easy way to avoid creating multiple entry points was eventually found, and the patch set was reposted as [`taskfd_send_signal()`](<https://lwn.net/ml/linux-kernel/20181206121858.12215-1-christian@brauner.io/>) with the same argument types. 

The submission tries to fix a problem that has been experienced by multiple people and the developers seem motivated to have the work done. The solution goes in the direction of following the long-established convention of using file descriptors. There is no conclusion yet if this approach will be accepted -- probably more iterations will still be needed. However, it seems likely that we will get an improvement in the robustness of signal usage in the not-that-far future.
