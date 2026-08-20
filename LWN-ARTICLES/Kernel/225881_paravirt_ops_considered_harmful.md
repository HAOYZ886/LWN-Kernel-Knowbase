---
title: "paravirt_ops considered harmful?"
url: https://lwn.net/Articles/225881/
date: "March 14, 2007"
category: paravirt ops
author: 
---

As flame wars go, this one was somewhat more technical and inscrutable than most. It was, however, still a flame war. The core issue was this: is the addition of the [paravirt_ops](<http://lwn.net/Articles/194543/>) layer, now beginning to be used to support running Linux under hypervisors, a good thing or a long-term maintenance disaster for the Linux kernel? 

It all started with a patch added to the -mm tree; it seems that some work on the new [clockevents](<http://lwn.net/Articles/223185/>) code broke the VMI virtualization layer. So the developers at VMware put together a fix, but that fix did not sit well with the core clockevents developers. In their view, it took much of the older time-related code, which they had worked so hard to get rid of, and shoved it back under the VMI layer. Thomas Gleixner [did not like](<https://lwn.net/Articles/225882/>) this solution: 

This is ugly as hell. NO_HZ enables the dyntick functions in idle(), irq_enter() and irq_exit() so the clockevents code is actually invoked. I have not looked close enough why this does work at all. I have the feeling that "working fine" means something like "does not explode". 

The right solution, according to Thomas, is for all of the people who are working on hypervisors and Linux to get together and come up with a single timer interface based on clockevents. This should not be all that hard of a job, in his opinion. The VMI hackers may well be willing to do that over time, but they don't see that as something which can be done in the near future. Their current code works, and, besides, they are on the verge of a product release and would rather not thrash things up at this time. 

"On the verge of a product release" is not an excuse which flies far on linux-kernel. This is doubly true in this case, where some of the people involved feel that the VMI developers should have seen clockevents coming and developed for that interface over the last year. They see the current VMI timer code as being the beginning of a long-term maintenance nightmare. 

Ingo Molnar [widened the discussion](<https://lwn.net/Articles/225884/>) to the problems he sees with paravirt_ops in general. The posting is long, but the core point seems to be this: every hypervisor connection implemented with paravirt_ops becomes an ABI that the kernel must then maintain forever. The paravirt_ops interface itself is supposed to insulate the kernel from changes, and that API can change. But each hypervisor interface done through paravirt_ops must continue to work into the future, meaning that certain sorts of fundamental design changes cannot be made. Maintaining compatibility with several hypervisors will be hard, and [Ingo sees bad things](<https://lwn.net/Articles/225887/>) when one inevitably breaks: 

And it doesn't matter whether we think that it was VMWare who messed up. Users/customers _will_ blame us: "v2.6.25 regresses, it wont run under ESX v1.12 anymore". Distro will yield and will undo whatever change breaks backwards compatibility with older hypervisors. (most likely it will be undone upstream already) Backwards compatibility acts as a very heavy barrier against certain types of paravirt_ops design changes. 

There have not been a whole lot of others supporting this point of view, though. The current abuses are seen as things which can be fixed, people seem to be sanguine about the ability to maintain compatibility in the paravirt_ops interface code, and, most likely, many people simply tune out of virtualization discussions. Linus [suggests](<https://lwn.net/Articles/225889/>) that Ingo point out specific problems (and fix them if he desires) rather than complaining about general problems. Ingo's response is that hypervisor interfaces should be treated like system calls, and added with the same degree of care and deliberation. 

In the end, it is not clear that anything will change. There is a high level of interest in getting hypervisor support into the kernel, and that process is unlikely to stop. So expect to see some more serious squabbles about what is done in hypervisor interfaces in the future. If we are lucky, that process, while noisy, will result in the evolution of the paravirt_ops code toward something which proves to be maintainable over the long term.
