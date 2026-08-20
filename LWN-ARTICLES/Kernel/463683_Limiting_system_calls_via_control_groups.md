---
title: "Limiting system calls via control groups?"
url: https://lwn.net/Articles/463683/
date: "October 19, 2011"
category: Control groups; Security; System calls
author: "By Jake Edge October 19, 2011"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
October 19, 2011

Limiting the system calls available to processes is fairly hot topic in the kernel security community these days. There have been several different proposals and the topic was [discussed](<https://lwn.net/Articles/458805/>) at some length at the recent Linux Security Summit but, so far, no solution has made its way into the mainline. Łukasz Sowa recently [posted an RFC](<https://lwn.net/Articles/463636/>) for a different mechanism to restrict syscalls, which may have advantages over other approaches. It also has a potential disadvantage as it uses a feature that is unpopular with some kernel hackers: control groups. 

Conceptually, Sowa's idea is pretty straightforward. An administrator could place a process or processes into a control group and then restrict which syscalls those processes (and their children) could make. The current interface uses system call numbers that are written to the `syscalls.allow` and `syscalls.deny` cgroup control files. Any system calls can be denied, but only those available to a parent cgroup could be enabled that way. Any process that makes a denied system call would get an `ENOSYS` error return. 

Using system call numbers seems somewhat painful (and those numbers are not the same across architectures), but may be unavoidable. But there are some other bigger problems, performance to begin with. Sowa reports 5% more system time used by processes in the root cgroup, which is a hefty penalty to pay. His patch hooks into the assembly language syscall fastpath, which is probably not going to fly. It is also architecture-specific and only implemented for x86 currently. Paul Menage [points out](<https://lwn.net/Articles/463741/>) that hooking into the `ptrace()` path may avoid those problems: 

Can't you hook into the ptrace callpath? That's already implemented on every architecture. Set the thread bit that triggers diverting to syscall_trace_enter() only when any of the thread's syscalls are denied, and then you don't have to work in assembly. 

Menage also mentions some other technical issues with the patch, but he is skeptical overall of the need for it. ""I'd guess that most vulnerabilities in a system can be exploited just using system calls that almost all applications need in order to get regular work done (open, write, exec ,mmap, etc) which limits the utility of only being able to turn them off by syscall number."" Because the approach only allows a binary on or off choice for the system calls, he doesn't necessarily think that it has the right level of granularity. The granularity argument echoes [the one made by Ingo Molnar](<https://lwn.net/Articles/332986/>) on a 2009 [proposal](<https://lwn.net/Articles/332438/>) to extend seccomp by adding a bitmask of allowed system calls. 

But there have been a number of projects that have expressed interest in having a more flexible seccomp-like feature in the kernel, starting with the Chromium browser team who have [proposed several ways](<https://lwn.net/Articles/450291/>) to do so. Seccomp provides a way to restrict processes to a few syscalls (`read()`, `write()`, `exit()`, and `sigreturn()`), but that is too inflexible for many projects. But Molnar has been very vocal in opposition to approaches that only allow binary decisions about system call usage, and he prefers a mechanism that [filters system calls](<https://lwn.net/Articles/441232/>) using Ftrace-style conditionals. That approach, however, is [not popular](<https://lwn.net/Articles/444745/>) with some of the other tracing and instrumentation developers. 

It is a quandary. There are a number of projects (e.g. QEMU, vsftpd, LXC) interested in such a feature, but no implementation (so far) has passed muster. Sowa's cgroup-based solution may well be yet another. Certainly the current performance for processes that are not in a cgroup (i.e. are in the root cgroup) is not going to be popular--an understatement--but even if Menage's suggestion (or some other mechanism) leads to a solution with little or no performance impact, there may be complaints because of the unpopularity of cgroups. 

There may be hope on the horizon in the form of a [proposed discussion](<https://lwn.net/Articles/450427/>) about expanding seccomp (or providing a means to disable certain syscalls) at the [upcoming Kernel Summit](<http://ksummit2011.kernel.org/>), though it does not seem to have made it onto the [agenda](<http://ksummit2011.kernel.org/agenda>). Certainly many of the participants in the mailing list discussions will be present. Control groups _is_ on the agenda, though, so there will be some discussion of that rather contentious topic. Look for LWN's coverage of the summit on next week's Kernel page.
