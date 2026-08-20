---
title: In search of an appropriate RLIMIT_MEMLOCK default
url: https://lwn.net/Articles/876288/
date: "November 19, 2021"
category: "Memory management-User-space memory locking; Resource limits; io uring"
author: "By Jonathan Corbet November 19, 2021"
---

By **Jonathan Corbet**  
November 19, 2021

One does not normally expect a lot of disagreement over a 13-line patch that effectively tweaks a single line of code. Occasionally, though, such a patch can expose a disagreement over how the behavior of the kernel should be managed. [This patch from Drew DeVault](<https://lwn.net/ml/linux-kernel/20211028080813.15966-1-sir@cmpwn.com/>), who is evidently taking a break from [stirring up the npm community](<https://drewdevault.com/2021/11/16/Cash-for-leftpad.html>), is a case in point. It brings to light the question of how the kernel community should pick default values for configurable parameters like resource limits. 

The kernel implements a set of resource limits applied to each (unprivileged) running process; they regulate how much CPU time a process can use, how many files it can have open, and more. The [`setrlimit()` man page](<https://man7.org/linux/man-pages/man2/setrlimit.2.html>) documents the full set. Of interest here is `RLIMIT_MEMLOCK`, which places a limit on how much memory a process can lock into RAM. Its default value is 64KB; the system administrator can raise it, but unprivileged processes cannot. 

Once upon a time, locking memory was a privileged operation. The ability to prevent memory from being swapped out can present resource-management problems for the kernel; if too much memory is locked, there will not be enough left for the rest of the system to function normally. The widespread use of cryptographic utilities like GnuPG eventually led to this feature being made available to all processes, though. By locking memory containing sensitive data (keys and passphrases, for example), GnuPG can prevent that data from being written to swap devices or core-dump files. To enable this extra security, the kernel community opened up the [`mlock()` system call](<https://man7.org/linux/man-pages/man2/mlock.2.html>) to all users, but set the limit for the number of pages that can be locked to a relatively low value. 

Uses of memory change over time. GnuPG does not really need more locked memory than it did years ago, but there are now other ways that users can run into the locked-memory limit. BPF programs, for example, are stored in unswappable kernel memory, with the space used being charged against this limit. These programs tend to be relatively small, but 64KB is likely to be constraining for many users. The big new consumer of locked memory, though, is [io_uring](<https://lwn.net/Articles/776703/>). 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Whenever the kernel sets up a user-space buffer for I/O, that buffer must be locked into memory for the duration of the operation. This locking is a short-lived affair and is not charged against the user's limit. There is, however, quite a bit of work involved in setting up an I/O buffer and locking it in memory; if that buffer is used for frequent I/O operations, the setup and teardown costs can reach a point where they slow the application measurably. As a way of eliminating this cost, the io_uring subsystem allows users to "register" their buffers; that operation sets up the buffers for I/O and leaves them in place where they can be used repeatedly. 

I/O buffers can be large, so locking them into memory can consume significant amounts of RAM; it thus makes sense that a limit on how much memory can be locked in this way should be imposed. So, when buffers are registered, the kernel charges them against the same locked-memory limit. This is where the 64KB limit becomes truly constraining; to make the use of io_uring worthwhile, one almost certainly wants to use much larger buffers than will fit in that space. The 64KB default limit, as a result, has the potential to make io_uring unavailable to users unless it is increased by distributors or administrators — and that tends not to happen. 

To avoid this problem, DeVault would like to raise that limit to 8MB. Expecting the problem to be addressed elsewhere, he said, is not realistic: 

> The buck, as it were, stops with the kernel. It's much easier to address it here than it is to bring it to hundreds of distributions, and it can only realistically be relied upon to be high-enough by end-user software if it is more-or-less ubiquitous. 

Matthew Wilcox [pointed out](<https://lwn.net/ml/linux-kernel/YZP6JSd4h45cyvsy@casper.infradead.org/>) that there are plenty of other ways for a malicious user to lock down at least 8MB of memory, so he saw no added danger from this change, but with a couple of reservations. Perhaps it would be better to somehow scale the limit, he said, so that it would be smaller on machines with small amounts of memory. He also wondered if 8MB was the right value for the new limit, or whether io_uring users would need still more. Jens Axboe, the maintainer of io_uring, [replied](<https://lwn.net/ml/linux-kernel/b97f1b15-fbcc-92a4-96ca-e918c2f6c7a3@kernel.dk/>) that ""8MB is plenty for most casual use cases"", and those are the cases that should "just work" without the need for administrator intervention. 

Andrew Morton, though, [was not convinced](<https://lwn.net/ml/linux-kernel/20211116133750.0f625f73a1e4843daf13b8f7@linux-foundation.org/>) about this value — or any other: 

> We're never going to get this right, are we? The only person who can decide on a system's appropriate setting is the operator of that system. Haphazardly increasing the limit every few years mainly reduces incentive for people to get this right. 

DeVault [answered](<https://lwn.net/ml/linux-kernel/CFRWRP3RTFT4.3VWRBA26OUSND@taiga/>) that ""perfect is the enemy of good"", and that he lacked the time to try to convince all of the distributors to configure a more realistic default. Morton's further [suggestion](<https://lwn.net/ml/linux-kernel/20211118135846.26da93737a70d486e68462bf@linux-foundation.org/>) that the limit should have been set to zero from the beginning to force a solution in user space was [not received well](<https://lwn.net/ml/linux-kernel/CFTL5A99FTIY.38WS1HS59BT2D@taiga/>). And that, more or less, is where the conversation wound down. 

One line of thought here seems to be that the kernel community should not try to come up with usable defaults for parameters like `RLIMIT_MEMLOCK`; that will force downstream distributors to think about what their users need and configure things accordingly. But that seems like a recipe for the status quo, where a useful new feature is, in fact, not useful on most systems. Putting some thought into reasonable default values is something one normally expects from a software project; it's not clear why the kernel would be different in this regard. So this change will, in all likelihood, eventually find its way in, but perhaps not until the emails-to-lines-changed ratio becomes even higher.
