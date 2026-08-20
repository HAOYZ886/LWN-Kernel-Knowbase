---
title: Toward a saner execve()
url: https://lwn.net/Articles/381158/
date: "March 30, 2010"
category: "Security-Security technologies; System calls-execve"
author: "By Jonathan Corbet March 30, 2010"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
March 30, 2010

Contemporary Linux systems allow processes to set up their environments in any of a number of ways. For various reasons, developers sometimes want even more flexibility; in particular, they would like to take something away (filesystem access, network access, capabilities) from a running process, usually in the name of security. The problem is that such changes can actually make security worse; as has been seen many times, privileged programs can be made to do strange and unfortunate things when run in unexpected environments. 

As Andy Lutomirski [notes](<http://lwn.net/Articles/380551/>), one response to this problem is to disable setuid semantics as well. But there are a lot of ways for the `execve()` system call to change a process's privileges which do not involve setuid programs; this is especially true in the presence of security modules. So Andy has proposed a different idea: opt out of `execve()` instead. To that end, he proposes a new `prctl()` option (`PR_RESTRICT_ME`) which could be used to add restrictions to a running process; the first of those is that the process cannot call `execve()`. Disabling `execve()` would be mandatory before any other restrictions could be added. 

But a process running in a restricted mode might still want to run other programs; that's how Linux programs often work. To accommodate that need, Andy has added a new system call, named `execve_nosecurity()`. This variant of `execve()` will run the indicated program, but it will perform absolutely no security transitions first. So no setuid, no SELinux type changes, etc. The end result is a system call with functionality similar to simply mapping the program into the caller's address space and running it directly. With `execve_nosecurity()`, it is not possible to increase privileges by running another program, so it should make the removal of capabilities from running processes safer. 

This patch should address a number of the concerns developers have had with the restricting of privileges. It's hard to tell for sure, though, because there has been very little in the way of response so far.
