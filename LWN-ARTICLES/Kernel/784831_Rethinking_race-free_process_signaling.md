---
title: "Rethinking race-free process signaling"
url: https://lwn.net/Articles/784831/
date: "April 4, 2019"
category: "System calls-pidfd send signal; pidfd"
author: "By Jonathan Corbet April 4, 2019"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
April 4, 2019

One of the new features in the 5.1 kernel is the `pidfd_send_signal()` system call. Combined with the (also new) [ability to create a file descriptor referring to a process](<https://lwn.net/Articles/773459/>) (a "pidfd") by opening its directory in `/proc`, this system call allows for the sending of signals to processes in a race-free manner. An extension to this feature proposed for 5.2 has, however, sparked a discussion that has brought the whole concept into question. It may yet be that the pidfd feature will be put on hold before the final 5.1 release while the API around it is rethought. 

The fundamental problem being addressed by the pidfd concept is process-ID reuse. Most Linux systems have the maximum PID set to 32768; if lots of processes (and threads) are created, it doesn't take a long time to use all of the available PIDs, at which point the kernel will cycle back to the beginning and start reusing the ones that have since become free. That reuse can happen quickly, and processes that work with PIDs might not notice immediately that a PID they hold referred to a process that has exited. In such conditions, a stale PID could be used to send a signal to the wrong process. As Jann Horn [pointed out](<https://lwn.net/ml/linux-kernel/CAG48ez2gb94SqS30Ai4+VBHhnzBp5Po9_u00nMrvUW6Wqq6hPA@mail.gmail.com/>), real vulnerabilities have resulted from this problem. 

A pidfd is a file descriptor that is obtained by opening a process's directory in the `/proc` virtual filesystem; it functions as a reference to the process of interest. If that process exits, its PID might be reused by the kernel, but any pidfds referring to that process will continue to refer to it. Passing a pidfd to `pidfd_send_signal()` will either signal the correct process (if it still exists), or return an error if the process has exited; it is guaranteed _not_ to signal the wrong process. So it would seem that this problem has been solved. 

#### Not so fast

In late March, Christian Brauner posted [a patch set](<https://lwn.net/ml/linux-kernel/20190329155425.26059-1-christian@brauner.io/>) adding another new system call: 

```
int pidfd_open(pid_t pid, unsigned int flags);
```

This system call will look up the given `pid` in the current namespace, then return a pidfd referring to it. This call was proposed to address cases where `/proc` is not mounted in a given namespace. For cases where `/proc` _is_ available, though, the patch set also implements a new `PIDFD_GET_PROCFD` `ioctl()` call that takes a pidfd, opens the associated process's `/proc` directory, and return a file descriptor referring to it. That descriptor, which functions as a pidfd as well, could then be used to read other information of interest out of the `/proc` directory. 

Linus Torvalds [had no fundamental problem](<https://lwn.net/ml/linux-kernel/CAHk-=wjq4dTPcz-qsvhpm5F42SDHCoqEWv1V=rs_kt6MC=ZThA@mail.gmail.com/>) with `pidfd_open()`, but he was rather less pleased with the `ioctl()` command. The core of his disagreement had to do with the creation of a second type of pidfd: one created by `pidfd_open()` would have different semantics than one created by opening the `/proc` directory or by calling `ioctl()`. In [his view](<https://lwn.net/ml/linux-kernel/CAHk-=wiaLtH41Mc5qAjOeCWavPqV0DhS581KYa0QBt8uraTK1Q@mail.gmail.com/>), either creation path should yield the same result on systems where `/proc` is mounted; there should be no need to convert between two types of pidfd. 

Brauner was not immediately accepting of that idea. He [worried](<https://lwn.net/ml/linux-kernel/20190331211041.vht7dnqg4e4bilr2@brauner.io/>) that the equivalence would force a dependency on having `/proc` enabled (a concern that Torvalds [dismissed](<https://lwn.net/ml/linux-kernel/CAHk-=wi3AE1-iRQ_7LOeSMNAMrGxRdC=gTjD30duVw4XRchcNQ@mail.gmail.com/>)), and that [it could expose information in `/proc`](<https://lwn.net/ml/linux-kernel/20190331220259.qntxynluk765hpnt@brauner.io/>) that might otherwise be hidden from a `pidfd_open()` caller. Torvalds [suggested](<https://lwn.net/ml/linux-kernel/CAHk-=wh0jgRkZiNdFD96Zpjx+_G+NVSHieAt+QgWCQBJ2A-5Aw@mail.gmail.com/>) tightening the security checks in that latter case. Even then, Andy Lutomirski [worried](<https://lwn.net/ml/linux-kernel/18C7FCB9-2CBA-4237-94BB-9C4395A2106B@amacapital.net/>), ""/proc has too much baggage"" to be made secure in this setting. It might be necessary, he said, to create a separate view of `/proc` that would be provided with pidfds. 

#### clone()

As the conversation went on, though, it became increasingly clear that `pidfd_open()` was not the end goal. That call is still racy — a PID could be reused in the time between when a caller learns about it and when the `pidfd_open()` call actually completes. There are ways of mitigating this problem, but it does still exist. The only truly race-free way of getting a reference to a process, it was agreed, is to create that reference as part of the work of creating the process itself. That means it should be created as part of a [`clone()`](<http://man7.org/linux/man-pages/man2/clone.2.html>) call. 

That could be made possible by adding a new flag (called something like `CLONE_PIDFD`) to `clone()` that would return a pidfd to the parent rather than a PID. There were some worries that `clone()` has run out of space for new flags, necessitating a new system call, but Torvalds indicated that there is still at least one bit available. As a result of the discussion, it seems likely that a patch implementing the new `clone()` behavior will be posted in the near future. 

That, however, leaves open the question of `pidfd_open()` and how pidfds should work in general. At one point, Brauner [suggested](<https://lwn.net/ml/linux-kernel/20190331150507.zpyugdvtmr6rgpda@brauner.io/>) breaking the connection with `/proc` entirely: a pidfd could be used for sending signals (or, in the future, waiting for a process), but its creation would not be tied to a `/proc` directory in any way. That would involve disabling the functionality in 5.1, something that can still be done since it is not yet part of an official kernel release. The problem of opening the correct `/proc` directory (to read information about the process) could be addressed by adding a field to the `fdinfo` file for the pidfd; the information there could be used to verify that a given `/proc` directory refers to the same process as the pidfd. 

It eventually became clear, though, that Torvalds instead favored retaining the tie between a pidfd and the `/proc` directory; he [called it](<https://lwn.net/ml/linux-kernel/CAHk-=wgjndAqzMBxgXZxbSRXLRqdXtNM3aHc9X-xj+Px1fsG-Q@mail.gmail.com/>) ""the most flexible option"". So, one day later, Brauner came back with [another plan](<https://lwn.net/ml/linux-kernel/20190401194214.4rbvmwogpke3cjcx@brauner.io/>): the connection with `/proc` would remain, but the `pidfd_open()` system call would be dropped since there would no longer be any real need for it. Should this plan be followed, which seems to be the most likely outcome, the existing 5.1 pidfd work could remain, since it is still a part of the final vision. 

If things play out this way, the new `clone()` option will likely appear in 5.2 or 5.3. Process-management systems that are concerned about races will then be able to use pidfds for safe process signaling. If nothing else, this discussion shows the value of having many developers looking at proposed API additions. In a setting where mistakes are hard to correct once they get out into the world, one wants to get things right from the outset if at all possible. 

#### A postscript

A contributing factor to the problem of PID reuse is the fact that the PID space is so small; for compatibility with ancient Unix systems (and the programs that ran on them), it's limited to what can be stored in a signed 16-bit value. That was a hard limit until the 2.6.10 release in 2002, when Ingo Molnar added a flexible limit capped at 4,194,304; the default limit remained (and remains) 32768, but it can be changed with the `kernel/pid_max` sysctl knob. 

At the time, Molnar placed a comment reading ""a maximum of 4 million PIDs should be enough for a while"" that endures to this day. Over 16 years later, it's clear that he was right. But as part of this discussion, Torvalds [said](<https://lwn.net/ml/linux-kernel/CAHk-=wiZ40LVjnXSi9iHLE_-ZBsWFGCgdmNiYZUXn1-V5YBg2g@mail.gmail.com/>) that perhaps the time has come to raise both the default and the limit. Setting the maximum PID to `MAXINT` would, he said, make a lot of the attacks harder. Whether such a change would break any existing software remains to be seen; it seems unlikely in 2019 but one never knows.
