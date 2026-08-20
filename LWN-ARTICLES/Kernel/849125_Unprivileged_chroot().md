---
title: Unprivileged chroot()
url: https://lwn.net/Articles/849125/
date: "March 15, 2021"
category: "System calls-chroot"
author: "By Jonathan Corbet March 15, 2021"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
March 15, 2021

It is probably fair to say that most Linux developers never end up using [`chroot()`](<https://man7.org/linux/man-pages/man2/chroot.2.html>) in an application. This system call puts the calling process into a new view of the filesystem, with the passed-in directory as the root directory. It can be used to isolate a process from the bulk of the filesystem, though its security benefits are somewhat limited. Calling `chroot()` is a privileged operation but, if Mickaël Salaün has his way with [this patch set](<https://lwn.net/ml/linux-kernel/20210310181857.401675-1-mic@digikod.net/>), that will not be true for much longer, in some situations at least. 

Typically, `chroot()` is used for tasks like "jailing" a network daemon process; should that process be compromised, its ability to access the filesystem will be limited to the directory tree below the new root. The resulting security boundary is not the strongest — there are a number of ways to break out of `chroot()` jails — but it can still present a barrier to attackers. `chroot()` can also be used to create a different view of the file system to, for example, run containers within. 

This system call is not available to just anybody; the `CAP_SYS_CHROOT` capability is required to be able to call `chroot()`. This restriction is in place to thwart attackers who would otherwise try to confuse (and exploit) setuid programs by running them inside a specially crafted filesystem tree. As a simple example, consider the sort of mayhem that might be possible if setuid programs saw a version of `/etc/passwd` or `/etc/sudoers` that was created by an attacker. 

The limitations of `chroot()` have long limited its applicability; in recent years it has fallen even further out of favor. [Mount namespaces](<https://lwn.net/Articles/689856/>) are a much more flexible mechanism for creating new views of the filesystem; they can also be harder to break out of. So relatively few developers see a reason to use `chroot()` for anything new. 

Thus, some folks were a bit surprised when Salaün showed up with his `chroot()` patch. Once applied, unprivileged processes are able to call `chroot()`, but only if a few conditions apply: 

  * The process in question must have done a [`prctl()`](<https://man7.org/linux/man-pages/man2/prctl.2.html>) call with the `PR_SET_NO_NEW_PRIVS` option. That prevents the process from gaining any new privileges; running setuid and setgid programs will no longer gain the privileges of the owner of the executable file, for example. Since privileged programs no longer exist in that mode, their privileges cannot be exploited. 
  * The process cannot be sharing its filesystem context ([`struct fs_struct`](<https://elixir.bootlin.com/linux/v5.11.6/source/include/linux/fs_struct.h#L9>), which contains the root and current working directories) with any other processes; otherwise the `chroot()` call would affect both processes, and the other one may not be expecting its filesystem environment to change abruptly. 
  * The new root must be underneath the current root in the filesystem hierarchy. This prevents trickery that could otherwise facilitate escape from an existing jail or mount namespace. 

If these conditions are met, it is argued, it is safe to allow a process to call `chroot()`. 

There is still the question of why one might _want_ to do that. Among other things, a functioning `chroot()` environment normally needs to have a minimally populated `/dev` directory; creating device nodes remains a privileged operation. And, as noted above, Linux has had better options than `chroot()` for some time now. But Salaün says that there are use cases where a process might want to sandbox itself after the things it needs from the wider environment (libraries, for example) have been loaded, and device files can often be done without. 

The initial reception for this patch has been a bit chilly at best. Eric Biederman [worried](<https://lwn.net/ml/linux-kernel/m1lfavt0bf.fsf@fess.ebiederm.org/>) about the security implications of unprivileged `chroot()` when mixed with other mechanisms: 

> Still allowing chroot after the sandbox has been built, a seccomp filter has been installed and no_new_privs has been enabled seems like it is asking for trouble and may weaken existing sandboxes. 

Casey Schaufler [argued](<https://lwn.net/ml/linux-kernel/4b9a1bb3-94f0-72af-f8f6-27f1ca2b43a2@schaufler-ca.com/>) that `chroot()` is obsolete and also worried about interactions: ""We're still finding edge cases (e.g. ptrace) where no_new_privs is imperfect"". He also [pointed out](<https://lwn.net/ml/linux-kernel/0dfd4306-8e7c-239b-2829-d4103395ea44@schaufler-ca.com/>) that access to `chroot()` is already finely controlled with the `CAP_SYS_CHROOT` capability: 

> CAP_SYS_CHROOT is specific to chroot. It doesn't give you privilege beyond what you expect, unlike CAP_CHOWN or CAP_SYS_ADMIN. Making chroot unprivileged is silly when it's possibly the best example of how the capability mechanism is supposed to work. 

Salaün has not answered all of these points, but seems undeterred; he posted [a second version of the patch set](<https://lwn.net/ml/linux-kernel/20210310181857.401675-1-mic@digikod.net/>) after that discussion had occurred. Without a stronger answer, though, upstreaming this change is likely to be difficult. Security-oriented developers will need some convincing that `chroot()` merits any improvements at all; the bar for changes that raise worries about unexpected interactions with other security mechanisms will be higher. 

The discussion is likely to come down to use cases in the end; is there truly a need for unprivileged `chroot()` in 2021? If there are users out there who could benefit from this feature, now would probably be a good time for them to come forward and explain why they need it. In the absence of that information, unprivileged `chroot()` seems likely to be one of those ideas that didn't quite make it.
