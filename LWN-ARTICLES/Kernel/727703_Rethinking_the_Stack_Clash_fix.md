---
title: Rethinking the Stack Clash fix
url: https://lwn.net/Articles/727703/
date: "July 13, 2017"
category: "Development model-User-space ABI; Security-Vulnerabilities"
author: "By Jonathan Corbet July 13, 2017"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
July 13, 2017

It has been nearly one month since the [Stack Clash vulnerability](<https://lwn.net/Articles/725832/>) was disclosed and some hardening measures were rushed into the 4.12 kernel release. Since then, a fair amount of work has gone into fixing problems caused by those measures and porting the result back to stable kernel releases. Now, it seems, the kernel developers are considering taking a different approach entirely. 

Stack Clash is generally seen as a user-space problem: a combination of large on-stack allocations and the lack of stack probing open up opportunities for an attacker to jump over the guard page at the end of the stack. Fixing those problems (and deploying the fixes widely) will take some time; meanwhile, it was thought, systems can be protected by replacing the kernel-provided guard page with a 1MB (or larger) guard region that, hopefully, cannot be jumped over. 

That guard region should be invisible to most programs, but it has created a surprising number of problems for some applications. A number of those issues have been worked around, but one has proved difficult to fix; unfortunately, that one is LibreOffice, which can crash on 32-bit systems when running Java components. The issue is a bit complicated but, in short, Java wants to enable code execution in memory located immediately below the stack, which runs afoul of the guard region. This, as Linus Torvalds [noted](<https://lwn.net/Articles/727705/>), is a problem: 

We really can't be breaking libreoffice. That's like a big classic no-no - it affects normal users that simply cannot be expected to work around it. For them, it's a "my office application no longer works" situation, and they just think the system is flaky. 

There is also the lingering fear that other, not-yet-discovered regressions lurk in user-space applications, regressions that might not surface until somebody does a kernel upgrade months or years from now. So, perhaps, adding a large guard region to work around a user-space problem is not the best answer. 

On most systems, the resource limit (rlimit) mechanism restricts the stack to a maximum of 8MB of memory. The hard limit tends to be much larger, though, so any unprivileged process can raise the effective stack limit to a larger value — as high as "infinite". The posted exploits for Stack Clash work by raising the stack rlimit, then running a setuid program with a vast array of arguments to fill that huge stack. If that program can be prodded into overflowing the stack without hitting the guard page, it may be possible to overwrite heap data beyond the stack and, from there, take over the program and make use of its privileges. 

There may be other ways to get a setuid program to exhaust its stack, but the command-line arguments method is easy and readily under the control of an attacker. (Note: the overflow is relatively easy; a successful exploit is harder). That suggests that the bulk of the Stack Clash exploits can be headed off by preventing the execution of setuid programs with a stack that's both (1) close to the heap area and (2) nearly full at the outset. 

Kees Cook took a stab at that problem with [this patch](<https://lwn.net/Articles/727274/>) attacking the first of those two points. The idea was that, when a process is about to run a setuid program, the stack limit should be reset to a reasonable value; the value Cook chose was whatever the `init` process has. This patch would not prevent stack exhaustion (indeed, it might cause it if there are setuid programs needing a huge stack), but it would keep the stack from growing large enough to impinge on a heap area. 

That patch didn't get far, though, since Torvalds [disliked it](<https://lwn.net/Articles/727707/>). One of his complaints was that special-casing setuid programs would be likely to lead to bugs or inadequate protection, since the relevant code would see relatively little use. So Cook's [next attempt](<https://lwn.net/Articles/727709/>) took a different tack: it places an upper bound on the amount of stack memory that can be occupied by a program's arguments at `exec()` time. In particular, that limit is 75% of the default stack limit, or 6MB, regardless of the current stack limit. This patch [has been merged](<https://git.kernel.org/linus/da029c11e6b12f321f36dac8771e833b65cec962>) for 4.13; it's not clear whether this change will find its way into the stable updates to earlier kernel releases. 

Limiting stack use by arguments should suffice to block a lot of attacks but, as it turns out, there's still a desire to enforce a limit on the size of the stack for setuid programs. One reason for that might be fears of some sort of pathological behavior that could be exploited to force a setuid program to overflow even a huge and mostly empty stack. But it also turns out that, if the stack rlimit is set to "infinity", the kernel will change the layout of a program's memory areas. A large stack limit suggests that the stack itself is likely to be large, so the kernel maps other memory areas low in the address space to preserve room for the stack to grow into. If, instead, the stack is limited, the kernel will map those areas at higher locations. As a result, the stack rlimit gives an attacker a bit of control over how the target program's memory is laid out — not a desirable thing to do. 

Thus [this patch series](<https://lwn.net/Articles/727714/>), which applies a maximum 8MB stack limit on setuid programs. These patches, posted on July 10, have not yet been merged; applying this limit required a surprising number of changes to the core `exec*()` system-call code, so more than the usual amount of review is indicated. There would appear to be general agreement on the goal, though, so this change seems likely to find its way into the mainline eventually. There has been some talk of allowing a larger stack via an annotation in the binary file, but that has not been implemented and may not be without a demonstrated need. 

At this point, nobody has said whether these changes will be enough to allow the removal of the larger guard region from the stack. Returning to the previous layout semantics would ease a lot of worries about regressions that might turn up months or years in the future, though, so it's not hard to see why the idea has appeal. It would seem that at least some of the kernel's internal memory-layout policies have become a part of the user-space ABI, so they need to be preserved if possible.
