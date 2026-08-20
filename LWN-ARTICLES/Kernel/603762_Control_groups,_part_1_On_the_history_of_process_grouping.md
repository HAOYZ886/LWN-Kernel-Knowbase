---
title: "Control groups, part 1: On the history of process grouping"
url: https://lwn.net/Articles/603762/
date: "July 1, 2014"
category: "Control groups-LWNs guide to"
author: "July 1, 2014 This article was contributed by Neil Brown Control groups"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

July 1, 2014

This article was contributed by Neil Brown

* * *

[Control groups](<https://lwn.net/Articles/604609/>)

While it may not be the most controversial feature ever added to Linux, there is little difficulty in finding mailing lists or Internet forums containing heated arguments about the merits of control groups — or even downright denials that the feature has any merit at all. Being bereft of a personal agenda on the matter or any deep understanding of the issues, I find it very hard to choose a side in these debates, which seriously lessens the enjoyment I can receive from them. As synthesizing a deep understanding is, I find, much more noble than synthesizing a personal agenda, and as having a discerning audience is an excellent motivation for thorough research, these articles are intended to help me and, hopefully, other readers to develop the deep understanding necessary to truly enjoy an informed debate on Linux control groups, which are also known as "cgroups".

To gain this understanding we will need both a broad perspective and some detailed analysis. The first two articles in this series will try to provide some perspective by first exploring the history of Unix to see what questions it raises about process groups, and then looking at hierarchies, both within and without the Unix family, to give us some yardsticks to measure the hierarchical aspects of cgroups. 

Subsequent articles will then delve into the nitty gritty details of cgroups and its various control subsystems and attempt to relate what we find to the questions and metrics the broader perspective gave us. 

#### Sixth Edition Unix

Unix has some history with process groupings and, more significantly, some evolution. Observing this change can help us to see important details. While it would be nice to start at the very beginning, a more practical starting point is the Sixth Edition of Unix, known hereafter as "V6 Unix". 

V6 Unix dates from the mid-1970s and was the first edition to get much exposure outside of Bell Labs. It supports two different groupings of processes, though to justify that we should first clarify what we mean by "a grouping of processes".

As in number theory, not every set is a group. The set of processes with a prime identifying number, for example, is certainly a set. However there is no mechanism in Unix (then or now) to distinguish these processes in any way from those with composite ID numbers. The remaining set of processes, with neither a prime nor a composite ID number, does have a distinctive behavior. As it contains only PID 1, though, it is hardly worth considering as a group. 

A number-theoretic group includes an operation that operates on members of the group with particular rules for what an "operation" is. For process groups, we will accept a much more vague concept and a different role for an "operation", but still there must be some operation within Unix which can affect, or be affected by, a particular process group.

A less facetious set than the "prime PID" set would be the set of processes owned by a given user ID (or "UID"). We won't consider this to be a group in V6 Unix because while there are operations (e.g. `kill`) that will affect processes in one group differently from processes in another group, there is no way to interact with the group as a whole.

The first set that really forms a meaningful group is the set of children of some given process. The only operation in V6 Unix which recognizes this group is the `wait()` system call and it can only detect if the group is empty or not empty. If `wait()` returns with error `ECHILD`, then the group is empty. If it returns without an error, or doesn't return, then the set wasn't empty when the call was made (though it might be empty when the call completes).

The same operation could be interpreted as applying to the set of descendants of the given process — that is, the children and any children of those children, etc. `ECHILD` is returned if and only if this set is empty too. This group has a significantly different behavior, though. In the group of children, a process cannot escape the group except by exiting. In the group of descendants, a process _can_ escape, if it is not an immediate child, when any ancestor of it exits.

Whether the ability to escape is a valuable property of groups or not depends, somewhat, on use-cases and expectations. In V6 Unix, the descendants of PID 1 (that set with a unity ID number) cannot escape but descendants of any other process can. This remained the case for variants of Unix and into Linux until Linux 3.4, when the `PR_SET_CHILD_SUBREAPER` option for [`prctl()`](<http://man7.org/linux/man-pages/man2/prctl.2.html>) was added. This allows a process to declare its group of descendant processes as closed so processes cannot escape. If any descendant dies, then all its children are inherited by the process which set this option.

The other, possibly more interesting, process grouping present in V6 Unix is determined by the `p_ttyp` field in the process structure (defined in [`proc.h`](<http://minnie.tuhs.org/cgi-bin/utree.pl?file=V6/usr/sys/proc.h>)), which is described as the "controlling tty". Whenever a process opens a "tty" device (see `dhopen()` in [`dh.c`](<http://minnie.tuhs.org/cgi-bin/utree.pl?file=V6/usr/sys/dmr/dh.c>)), which would be a serial data connection to a teletype or similar terminal, then if this field is not already set, it will be set to point to the newly opened device. The field is also inherited over a `fork()` or `exec()`, so once a process gained a controlling tty, that would continue to apply to the process and all of its future descendants.

One effect of `p_ttyp` is that any I/O to `/dev/tty` will go to the controlling tty, but this doesn't really qualify as a "group" operation, as it affects individual processes separately. The "group" operations for controlling ttys involve the delivery of signals (see `signal()` in [`sig.c`](<http://minnie.tuhs.org/cgi-bin/utree.pl?file=V6/usr/sys/ken/sig.c>)). If a `DEL` or `FS` (`control-\`) character is typed on a tty, then the signal `SIGINT` or `SIGQIT` is sent to all processes in the group that have that tty as their controlling tty. Similarly if a disconnect event is detected (like a modem hanging up), a `SIGHUP` is sent to the same group of processes. Signals can also be sent with the `kill()` system call. An attempt to send a signal to PID 0 will send it to every process with the same controlling tty as the sending process.

It is quite reasonable to think of this grouping as a prototype of cgroups. It is clearly about the grouping of processes and clearly about controlling those processes -- though only through sending a signal. These groups are created automatically, based on behavior, and are permanent -- once in a group, the process cannot escape. It appears that they were not perfect, though. The next edition brought changes.

#### Seventh Edition Unix

While V6 Unix supported process groups, it did not use that terminology. V7 Unix did, and had a richer concept of _group_. The `p_ttyp` field still exists, though its role was restricted to managing `/dev/tty` access. It was renamed to `u_ttyp` and moved to `struct user` ([`user.h`](<http://minnie.tuhs.org/cgi-bin/utree.pl?file=V7/usr/sys/h/user.h>)) — a structure that could be swapped out to disk with the rest of the process. `struct proc` ([`proc.h`](<http://minnie.tuhs.org/cgi-bin/utree.pl?file=V7/usr/sys/h/proc.h>)) instead had a new `p_pgrp` field to manage process groups. It was set on the first `open()` of a tty and used for delivering `SIGINT`, `SIGQUIT` (which has now gained a 'U'), and `SIGHUP`, and for delivering signals sent to PID 0\. But V7 also brought more flexibility.

The key change was that process groups now had an independent identity and an independent name — independent of the tty, at least. When a process without a controlling tty first opened a tty, a new process group would be created with an ID number matching the process ID number of that process. Though the ID was copied, it really was a new ID for a new object. The group can continue to exist even if the original process exits. Any remaining children will keep the group active and prevent the ID number from being reused, either as a process-group ID or as a process ID.

One consequence of this is that if you log off a tty and log back on again, you get a new process group, and the `t_pgrp` field in the `struct tty` structure will be changed. Unlike the situation in V6 Unix, a signal sent to a process group will never go to a process from a previous login on the same tty.

Another consequence is that process groups could be used for more than just ttys. Seventh Edition Unix had a "multiplexor driver" (`mpxchan` in [`mx1.c`](<http://minnie.tuhs.org/cgi-bin/utree.pl?file=V7/usr/sys/dev/mx1.c>) and [`mx2.c`](<http://minnie.tuhs.org/cgi-bin/utree.pl?file=V7/usr/sys/dev/mx2.c>)) which, though short-lived, still leaves a legacy in the current [`stat()`](<http://man7.org/linux/man-pages/man2/stat.2.html>) manual page:

```
3000   S_IFMPC         030000   multiplexed character special (V7)
    [...]
    7000   S_IFMPB         070000   multiplexed block special (V7)
```

The multiplexor worked a little bit like a socket interface and allowed different processes to connect to each other. An interface was available to form a separate process group for several interconnecting processes, so the master could send a signal to all other members of the group.

V7 Unix process groups were still closed, with processes generally unable to leave them. `mpxchan` does appear to allow a process to leave its original process group to join a group for a multiplexed channel, but it isn't clear that this was an intended consequence.

#### Fourth Berkeley Software Distribution

It is a bit of a large jump from V7 to 4BSD, having at least Unix 32v and 3BSD in [the meantime](<http://en.wikipedia.org/wiki/File:Unix_history-simple.svg>). But this is, to some extent, a personal journey and 4.3BSD was the next release that I used.

In 4BSD, we find that a lot has happened with process groups. In 4.3BSD, the set of processes with the same UID has become a group, in that a signal can be sent to all processes in that set (see `kill()` in [`kern_sig.c`](<http://minnie.tuhs.org/cgi-bin/utree.pl?file=4.3BSD/usr/src/sys/sys/kern_sig.c>)). Sending a signal to a PID of -1 will deliver it to all processes with the same UID as the sending process (though, if sent from a privileged process, the signal will be sent to every process regardless of UID). More significant is that by 4.4BSD there was now a limited hierarchical structure to process groups.

One of the many innovations in the Berkeley versions of Unix was "[job control](<http://en.wikipedia.org/wiki/Job_control_%28Unix%29>)". A "job" here refers to one or more processes working together on a particular task. Unix already had the ability to put some jobs in the "background", but it was implemented in a fairly _ad hoc_ manner. Such processes would be told to ignore any signals from the user (`SIGINT` and `SIGQUIT` would both be set to `SIG_IGN` before starting the process) and the shell would simply not wait for those processes to finish. This mostly worked well, but once a job was in the background, it had to stay there. Also, if such a process wrote to the terminal, its output could get mingled with output from foreground processes, resulting in a mess.

With BSD "job control", each job is placed into its own process group and the shell can tell the terminal to change its idea of which is the current foreground job (and so would receive signals and input and could generate output), and which jobs are in the background so they should be isolated.

The pre-existing concept that process groups were essentially per-login was still important, if only to provide a degree of compatibility with "System V" Unix, a separate path of development from AT&T. In 4.4BSD, these per-login process groups were re-introduced as "sessions". Each process ([`proc.h`](<http://minnie.tuhs.org/cgi-bin/utree.pl?file=4.4BSD/usr/src/sys/sys/proc.h>)) was (potentially) a member of a process group. Each process group was a member of a session. Each terminal ([`tty.h`](<http://minnie.tuhs.org/cgi-bin/utree.pl?file=4.4BSD/usr/src/sys/sys/tty.h>)) had a foreground process group, `t_pgrp`, and a controlling session, `t_session`.

Sessions were, and are, much like the V7 Unix process groups, though there are differences. One is that it is not possible to send a signal to all processes in a given session: that functionality only works for process groups, which are now per-job. Another is that a process can leave its session and create a new one by simply calling the [`setsid()`](<http://man7.org/linux/man-pages/man2/setsid.2.html
>) system call.

Either of these are sufficient to frustrate the task of killing all processes at logout — as local policy required in student labs a long time ago in a career far away. A frustration which was, at the time, unfixable due to a dependence on closed-source kernels.

On a modern, windowed desktop, these sessions and process groups are still present, but don't mean quite what they once meant. It is fairly easy to see how session IDs and process-group IDs are assigned by displaying the `sess` and `pgrp` fields with `ps`, as follows:

```
ps -axgo comm,sess,pgrp
```

There is no longer a well defined process grouping for a login session. Instead, each terminal window gets its own session, as do various other applications if they were written to request one. Each job started from the shell prompt still gets its own process group, but there is much less need to start and stop these jobs — rather than suspending the currently running job in one terminal window, it is just as easy to pop up another window and run some new command there.

To properly represent the groupings of processes relevant for a modern desktop, we really need a deeper hierarchy. One level would represent the login sessions, one would represent the applications running in those sessions, and one could be used for jobs within an application. The sessions and process groups that Linux inherits from 4.4BSD can give us only two of those levels. Maybe we can look to cgroups for the third.

#### Issues

Reflecting on these changes and experiences with process groups, there are a number of issues that may be worth considering when trying to form an opinion on the more modern form of cgroups:

  * Names for groups: In V6 Unix the only name was that of the associated resource: a tty. This changed to be an ID number in the same namespace as process ID numbers. In retrospect this sharing of namespaces might seem a little clumsy, though it was clearly convenient. As the kernel was solely responsible for allocating names (another noteworthy feature), any clumsiness remained safely inside that kernel.

  * Overlapping uses. The same mechanism was originally used to guide both the delivery of signals and the processing of I/O to `/dev/tty`. These were quickly separated since they are clearly related, but are not identical.

  * Should a process be able to escape its containing group? We have seen a progression in the answer to this, from "no" to "yes". Having the flexibility can be useful in some cases, but having control can be useful in others. Being able to enter a different job under the same session is easy to defend. Being able to create a new session is not so obviously useful for an unprivileged process.

  * What role does a hierarchy play? Process groups have only gained even a limited hierarchy toward the end of their development. Is this important? How can it be used? 

That last point, hierarchy, certainly is important. A lot of the recent changes in cgroups, and a significant part of the disagreements, relate to hierarchy. While the history of process groups has given us a glimpse of hierarchy it is not enough to develop any real understanding. For that we will need to look elsewhere. In the next installment we will examine a few different "elsewheres" to develop a perspective on hierarchy that we will then take to the inner details of cgroups to see if the former can help us to better understand the latter.
