---
title: Adding the pidfd abstraction to the kernel
url: https://lwn.net/Articles/801319/
date: "October 7, 2019"
category: pidfd
author: "By Jonathan Corbet October 7, 2019 Kernel Recipes"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
October 7, 2019

* * *

[Kernel Recipes](<https://lwn.net/Archives/ConferenceByYear/#2019-Kernel_Recipes>)

One of the many changes in the 5.4 kernel is the completion (insofar as anything in the kernel is truly complete) of the [pidfd API](<https://lwn.net/Articles/794707/>). Getting that work done has been "a wild ride so far", according to its author Christian Brauner during [a session](<https://kernel-recipes.org/en/2019/talks/pidfds-process-file-descriptors-on-linux/>) at the 2019 Kernel Recipes conference. He went on to describe the history of this work and some lessons for others interested in adding major new APIs to the Linux kernel. 

A pidfd, he began, is a file descriptor that refers to a process — or, more correctly, to a process's thread-group leader. There do not appear to be any use cases for pidfds that refer to an individual thread for now; such a feature could be added in the future if the need arises. Pidfds are stable (they always refer to the same process) and private to the owner of the file descriptor. Internally to the kernel, a pidfd refers to the [`pid` structure](<https://elixir.bootlin.com/linux/latest/source/include/linux/pid.h#L18>) for the target process. Other options (such as `struct task_struct`) were available, but that structure is too big to pin down indefinitely (which can be necessary, since a pidfd can be held open indefinitely). 

Why did the kernel need pidfds? The main driving force was the problem of process-ID (PID) recycling. A process ID is an integer, drawn from a (small by default) pool; when a process exits, its ID will eventually be recycled and assigned to an entirely unrelated process. This leads to a number of security [![\[Christian
Brauner\]](https://static.lwn.net/images/conf/2019/kr/ChristianBrauner-sm.jpg)](<https://lwn.net/Articles/801323/>) issues when process-management applications don't notice in time that a process ID has been reused; he put up a list of CVE numbers (visible in [his slides [SlideShare]](<https://www.slideshare.net/ennael/kernel-recipes-2019-pidfds-process-file-descriptors-on-linux>)) for vulnerabilities resulting from PID reuse. There have been macOS exploits as well. It is, he said, a real issue. 

Beyond that, Unix has long had a problem supporting libraries that need to create invisible helper processes. These processes, being subprocesses of the main application, can end up sending signals to that application or showing up in [`wait()`](<http://man7.org/linux/man-pages/man2/waitpid.2.html>) calls, creating confusion. Pidfds are designed to allow the creation of this kind of hidden process, solving a persistent, difficult problem. They are also useful for process-management applications that want to delegate the handling of specific processes to a non-parent process; the Android low-memory killer daemon (LMKD) and systemd are a couple of examples. Pidfds can be transferred to other processes by the usual means, making this kind of delegation possible. 

Brauner said that a file-descriptor-based abstraction was chosen because it has been done before on other operating systems and shown to work. Dealing with file descriptors is a common pattern in Unix applications. 

There are, he said, quite a few user-space applications and libraries that are interested in using pidfds. They include D-Bus, Qt, systemd, checkpoint-restore in user space (CRIU), LMKD, bpftrace, and the Rust "mio" library. 

#### Implementing pidfds

Brauner said that he started by looking at what other operating systems have done. He made a mistake, though, by not looking at how other systems implemented this feature until after he had gotten code of his own written. Illumos has an API — `procopen()` and friends — that is implemented in user space. Neither OpenBSD nor NetBSD has a pidfd implementation at all, but FreeBSD does in the form of its [process file descriptors](<https://www.freebsd.org/cgi/man.cgi?query=pdfork&sektion=2>). The idea is the same, but that implementation differs in the details. 

There have been previous attempts to add this idea to Linux as well, he said. These include a `forkfd()` system call and the [`CLONE_FD` flag](<https://lwn.net/Articles/636646/>) for `clone()`. None of these made it in; Brauner looked at them to try to figure out why. The `CLONE_FD` idea in particular tried to do too many things at once, he said. 

In an attempt to avoid a similar fate, Brauner did the pidfd work over the course of four kernel releases. That gave him (and the community) plenty of time to think about how the various parts of the API should work. The first piece that he bit off was [sending signals to processes](<https://lwn.net/Articles/784831/>) in a race-free way; it was "the obvious use case", he said. People had a lot of ideas about how this feature should work, so focusing the discussion was a bit of a challenge. These ideas included using `/proc` files, new `ioctl()` calls, and more; they were all aimed at the signaling problem in particular, but he had a more general API in mind from the beginning. In the end, `pidfd_send_signal()` went into 5.1. 

There was still a race condition involved, though, since a pidfd had to be obtained for a process after the process had been created. The answer was to return a pidfd directly from `[clone()](<http://man7.org/linux/man-pages/man2/clone.2.html>)`. There was some uncertainty about just what should be returned, though; should it be a file descriptor referring to a `/proc` file or something else? In the end, he sent two separate RFC patch postings, one using `/proc` and one using anonymous inodes instead. The `/proc` version was "nasty", he said, and would have probably led to an eventual need to rework procfs. After seeing the two ideas, a consensus formed around using anonymous inodes. 

One important design decision, he said, was to mark each pidfd to be closed by default on [`execve()`](<http://man7.org/linux/man-pages/man2/execve.2.html>) calls. He didn't want to see pidfds being leaked into unrelated applications. 

Returning a pidfd from `clone()` was added in 5.2. That work left Brauner feeling a little guilty, though, since he used the last available `clone()` flag bit for `CLONE_PIDFD`. That led to the implementation of [`clone3()`](<https://lwn.net/Articles/792628/>), which has a dedicated return argument for a pidfd. 5.3 also saw the addition of polling support for pidfds; this is important since it will be the main way to return an exit status to non-parent processes. [`pidfd_open()`](<https://lwn.net/Articles/789023/>) was also added in 5.3; it allows the creation of a pidfd for an existing process. 

In 5.4, the `waitid()` system call gained a new `P_PIDFD` flag, allowing a process to wait on a pidfd directly. That essentially completes the pidfd API as it had been originally envisioned. 

#### Future work and lessons

Like any other kernel API, pidfds will continue to evolve over time, Brauner said. One feature he would like to add is sending a `SIGKILL` signal to a process when the last pidfd referring to it is closed. That is something FreeBSD supports now, but Linux will need to do things a bit differently. When a FreeBSD `close()` call returns, all of the work in the kernel is done; Linux, instead, can defer work to a workqueue to be done asynchronously later. Thus, the process may continue to exist for a while after that last `close()` call returns, which may not be what the application expects. He has a proof-of-concept implementation of how this feature could work in Linux, but he's not entirely happy with it yet. 

Another upcoming feature is "exclusive waiting": marking a process so that only a pidfd can be used to wait for it. In other words, an ordinary `wait()` call (or any of its variants) will not return such a process's exit status, which will go only to a `waitid()` call that provides the right pidfd. This feature is aimed at the "invisible helper process use case". We probably want it, he said, but he still has to work out all of the semantics for it. 

Pidfds also need to be better integrated with the namespace mechanism. One potentially useful feature would be to pass a pidfd to `[setns()](<http://man7.org/linux/man-pages/man2/setns.2.html>)`; the result would be to enter several namespaces simultaneously. That is not something that can be done on current Linux systems. He is also thinking about adding a socket option to get a pidfd rather than an ordinary process ID for the peer on a local connection. 

Brauner concluded with the lessons he has learned from this work. The first is that "speed matters". But, in this case, he was not arguing for going as fast as possible; instead he recommends picking a sustainable speed for the addition of new features. That will give time to respond to people and get things right the first time. Developers should, he said, be open about what they don't know; that encourages other developers to help out. In this case, he got help from a number of senior kernel developers while implementing pidfds. Finally, he said, "be resilient" in the face of reviews. He felt that he "looked dumb" after the first posting of `pidfd_send_signal()`, but he is glad he pushed through that experience and got the work into the kernel. 

[Your editor thanks the Linux Foundation, LWN's travel sponsor, for supporting his travel to this event.]
