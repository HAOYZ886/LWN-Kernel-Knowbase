---
title: RLIMIT_NPROC and setuid()
url: https://lwn.net/Articles/451985/
date: "July 20, 2011"
category: "Security-Vulnerabilities"
author: "By Jake Edge July 20, 2011"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
July 20, 2011

The `setuid()` system call has always been something of a security problem for Linux (and other Unix systems). It interacts oddly with security and other kernel features (e.g. the unfortunately named "[sendmail-capabilities bug](<http://userweb.kernel.org/~morgan/sendmail-capabilities-war-story.html>)") and is often used incorrectly in programs. But, it is part of the Unix legacy, and one that will be with us at least until the 2038 bug puts Unix systems out of their misery. A recent patch from Vasiliy Kulikov arguably shows these kinds of problems in action: weird interactions with resource limits coupled with misuse of the `setuid()` call. 

There is a fair amount of history behind the problem that Kulikov is trying to solve. Back in 2003, programs that used `setuid()` to switch to a non-root user could be used to evade the limit on the number of processes that an administrator had established for that user (i.e. `RLIMIT_NPROC`). But that was fixed with a [patch from Neil Brown](<http://lkml.org/lkml/2003/7/13/226>) that would cause the `setuid()` call to fail if the new user was at or above their process limit. 

Unfortunately, many programs do not check the return value from calls to `setuid()` that are meant to reduce their privileges. That, in fact, was exactly the hole that sendmail fell into when Linux capabilities were introduced, as it did not check to see that the change to a new UID actually succeeded. Buggy programs that don't check that return value can cause fairly serious security problems because they assume their actions are limited by the reduced privileges of the switched-to user, but are actually still operating with the increased privileges (often root) that they started with. In effect, the 2003 change made it easier for attackers to cause `setuid()` to fail when `RLIMIT_NPROC` was being used. 

Kulikov [described](<https://lwn.net/Articles/452037/>) the problem back in June, noting that it was not a bug in Linux, but allowed buggy privileged programs to wreak havoc: 

I don't consider checking RLIMIT_NPROC in setuid() as a bug (a lack of syscalls return code checking is a real bug), but as a pouring oil on the flames of programs doing poorly written privilege dropping. I believe the situation may be improved by relatively small ABI changes that shouldn't be visible to normal programs. 

In the posting, he suggested two possible solutions to the problem. The first is to move the check against `RLIMIT_NPROC` from `set_user()` (a `setuid()` helper function) to `execve()` as most programs _will_ check the status of that call (and can't really cause any harm if they don't). The other suggestion is one that was [proposed](<https://lkml.org/lkml/2006/8/19/129>) by Alexander Peslyak (aka Solar Designer) in 2006 to cause a failed `setuid()` call to send a `SIGSEGV` to the process, which would presumably terminate those misbehaving programs. 

The first solution is not complete because it would still allow users to violate their process limit by using programs that do a `setuid()` that is not followed by an `execve()`, but that is a sufficiently rare case that it isn't considered to be a serious problem. Peslyak's solution was seen as too big of a hammer when it was proposed, especially for programs that _do_ check the status of `setuid()`, and might have proper error handling for that case. 

There were no responses to his initial posting, but when he brought it back up on July 6, he was [pleasantly surprised](<https://lwn.net/Articles/452047/>) to get a [positive response](<https://lwn.net/Articles/452178/>) from Linus Torvalds: 

My reaction is: "let's just remote the crazy check from set_user() entirely". If somebody has credentials to change users, they damn well have credentials to override the RLIMIT_NPROC too, and as you say, failure is likely a bigger security threat than success. 

The whole point of RLIMIT_NPROC is to avoid fork-bombs. If we go over the limit for some other reason that is controlled by the super-user, who cares? 

That led to the [patch](<https://lwn.net/Articles/452034/>), which changed `do_execve_common()` to return an error (`EAGAIN`) if the user was over their process limit and removed the check from `set_user()`. The patch was generally well-received, though several commenters were not convinced that it should go into the -rc for 3.0 as Torvalds had suggested. In fact, as Brown dug into the patch, he [saw a problem](<https://lwn.net/Articles/452105/>) that might need addressing: 

Note that there is room for a race that could have unintended consequences. 

Between the 'setuid(ordinary-user)' and a subsequent 'exit()' after execve() has failed, any other process owned by the same user (and we know where are quite a few) would fail an execve() where it really should not. 

Basically, the problem is that switching the process to a new user could now exceed the process limit, but that limit wouldn't actually be enforced until an `execve()` was done (the failure of which would presumably cause the process to exit). In the interim, any `execve()` from another of the user's processes would fail. It's not clear how big of a problem that is, though it could certainly lead to unexpected behavior. Brown offered up a [patch](<https://lwn.net/Articles/452112/>) that would address the problem by adding a process flag (`PF_NPROC_EXCEEDED`) that would be set if a `setuid()` caused the process to exceed `RLIMIT_NPROC` and would then be checked in `do_execve_common()`. Thus, only the `execve()` in the offending process would fail. 

Kulikov and Peslyak liked the approach, though Peslyak was not convinced it added any real advantages over Kulikov's original patch. He also [pointed out](<https://lwn.net/Articles/452113/>) that there could be a indeterminate amount of time between the `setuid()` and `execve()`, so the `RLIMIT_NPROC` test should be repeated when `execve()` is called: ""It would be surprising to see a process fail on execve() because of RLIMIT_NPROC when that limit had been reached, say, days ago and is no longer reached at the time of execve()."" 

So far, Brown has not respun the patch to add that test. There is also the question of whether the problem that Brown is concerned about needs to be addressed at all, and whether it is worth using up another process flag bit (there are currently only three left) to do so. In the end, some kind of fix is likely to go in for 3.1 given Torvalds's interest in seeing this problem with buggy programs disarmed. It's unclear which approach will win out, but either way, `setuid()` will not fail due to exceeding the allowable number of processes. 

As Kulikov and others noted, it is definitely _not_ a bug in the kernel that is being fixed here. But, it is a common enough error in user-space programs--often with dire consequences--which makes it worthwhile to fix as a [pro-active security measure](<https://lwn.net/Articles/451405/>). Peslyak [listed](<https://lwn.net/Articles/452139/>) several recent security problems that arose from programs that do not check the return value from `setuid()`. He also noted that the problem is not limited to setuid-root programs, as other programs that try to switch to a lesser--differently--privileged user can also cause problems when using `setuid()` incorrectly. 

The impact of this fix is quite small, and badly written user-space programs--even those meant to run with privileges--abound, which makes this change more palatable than some other pro-active fixes. As we have seen before, `setuid()` is subtle and quick to anger; it can have [surprising interactions](<https://lwn.net/Articles/368730/>) with other seemingly straightforward security measures. Closing a hole with `setuid()`, even if the problem lives in user space, will definitely improve overall Linux security.
