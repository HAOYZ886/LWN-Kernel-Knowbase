---
title: Unscheduled maintenance for sched.h
url: https://lwn.net/Articles/713890/
date: "February 8, 2017"
category: Include files
author: "By Jonathan Corbet February 8, 2017"
---

By **Jonathan Corbet**  
February 8, 2017

The kernel contains a large number of header files used to declare data structures and functions needed in more than one source file. Many are small and only used in a few places; others are large and frequently included. Header files have a tendency to build up over time since they often do not get as much attention as regular C source files. But there can be costs associated with bloated and neglected header files, as a current project to clean up one of the biggest ones shows. 

The 0.01 kernel release contained a grand total of 31 header files, nine of which lived in `include/linux`. One of those, [`<linux/sched.h>`](<https://lwn.net/Articles/713897/>), weighed in at all of 230 lines — the largest header file in that directory. Things have changed just a little bit since then. The upcoming 4.10 kernel contains 18,407 header files, just under 10,000 of which are intended for use outside of a specific subsystem. The [4.10 version of `<linux/sched.h>`](<https://lwn.net/Articles/713896/>) is 3,674 lines, but that understates its true weight: it directly includes 50 other header files, many of which will have further includes of their own. This is not the 0.01 `<linux/sched.h>` anymore. 

Ingo Molnar has [decided](<https://lwn.net/Articles/713712/>) that it is time to do something about this header file. A large header has its costs, especially when it is (by your editor's count) directly included into 2,500 other files in the kernel. An extra 1,000 lines of bloat expands into 2.5 million lines more code that must be compiled in a (full) kernel build, slowing compilation times significantly. A large and complex header file is also difficult to maintain and difficult to change; there are too many subtle dependencies on it throughout the kernel. 

How did this file get into this condition? As Molnar put it: 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

The main reason why it's so large is that since Linux 0.01 it had been the Rome of the kernel: all headers lead to it, due to almost every kernel subsystem having fields embedded in task_struct. sched.h has to know about the various structure definitions of various kernel subsystems - even if the scheduler never makes direct use of 90% of those fields. 

Molnar's response is a petite 89-part patch set intended to disentangle the `sched.h` mess. It starts by splitting out many of the more esoteric scheduler interfaces that are not needed by most users of `<linux/sched.h>`. This header is often included by driver code, which typically needs a small subset of the available interfaces, but which has no use for CPU frequency management, CPU hotplugging, accounting, or many other scheduler details. Code that needs the more specialized interfaces can find a set of smaller header files under `include/linux/sched`, but, Molnar says, 90% of users have no need for those other files. 

Beyond the split-up, the patch set cleans up the interfaces with a number of other ""entangled"", heavily-used header files so that each can be included separately. That eliminates the need to include those headers in `sched.h`. There was also a certain amount of historical cruft: header files that may have been needed at one time, but which were never removed from `sched.h` when that need went away. 

The result is a leaner `sched.h` that, Molnar says, can save 30 seconds on an `allyesconfig` kernel build. There are some details to be taken care of, though, beyond fixing source files that need the interfaces that have been split out to their own files. Since `sched.h` included so many other files, code that included it could get away without including the others, even if it needed them. Kernel code is supposed to explicitly include every header it needs and not rely on secondary inclusions but, if the code compiles anyway, it is easy to overlook a missing `#include` line. Taking those inclusions out of `sched.h` meant fixing up code elsewhere in the kernel that stopped compiling. 

After this work is done, the resulting patch set touches nearly 1,200 files; it is not a lightweight change, in other words. Molnar suggested that the patch set should be applied at the end of the merge window in the hope of minimizing the effects on other outstanding patches. He did not specify _which_ merge window he was targeting; 4.11 might still be possible and might be as reasonable a choice as any. Most patch sets are expected to spend some time in linux-next for wider testing, but this set almost certainly cannot go there without creating a massive patch-conflict nightmare. 

There are some changes that will need to be made before this work can be merged, though. Linus Torvalds [liked the end result](<https://lwn.net/Articles/713910/>), but was not pleased with how the patch set is organized. The changes are mixed together in a way that makes the patches hard to review and which, as was seen in a couple of cases, makes it easy for mistakes to slip in. 

He suggested that, instead, the series should start by splitting out parts of `sched.h`, but leaving things externally the same by including the split-out files back into `sched.h`. These changes could thus be made without changing code elsewhere in the kernel. After that, the back-includes could be removed, one by one, with the necessary fixes being applied elsewhere. The patches in this part of the series would consist of only `#include` changes and would, thus, be quick to review and verify. Molnar [agreed](<https://lwn.net/Articles/713911/>) to rework the patches along these lines, though he warned that this work ""will increase the patch count by at least 50%"". Making the patch set easier to review (and to bisect) will, hopefully, more than make up for the increased patch count. 

If this work can be completed in a convincing way before the close of the merge window, it may well make sense to apply it right away, even though the combination of big, intrusive, and new normally suggests that it may be better to wait. Causing this work to sit out for another development cycle would force much of it to be redone, and the end result may not be any more "ready" in 4.12 than it would be for 4.11. Of course, once this patch set is merged and the final loose ends tied down, the work is not yet done; there are a number of other large and messy header files in the kernel tree. The next target for a split-up may be another huge header file present since the 0.01 release: `<linux/mm.h>`.
