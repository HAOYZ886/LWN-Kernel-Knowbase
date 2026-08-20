---
title: "kgdb getting closer to being merged?"
url: https://lwn.net/Articles/270089/
date: "February 20, 2008"
category: "Development tools-kgdb; kgdb"
author: "By Jake Edge February 20, 2008"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
February 20, 2008

The kernel source level debugger, kgdb, has been around for a long time, but never in the mainline tree. Linus Torvalds is not much of a fan of debuggers in general and has always resisted the inclusion of kgdb. That looks like it might be changing somewhat, with the inclusion of kgdb in 2.6.26 now a distinct possibility. 

Over the years, Torvalds has made various pronouncements about debuggers, particularly kernel debuggers, a [long message to linux-kernel in 2000](<http://lwn.net/2000/0914/a/lt-debugger.php3>) seems to outline his objections: 

I happen to believe that not having a kernel debugger forces people to think about their problem on a different level than with a debugger. I think that without a debugger, you don't get into that mindset where you know how it behaves, and then you fix it from there. Without a debugger, you tend to think about problems another way. You want to understand things on a different _level_. 

An attempt to sneak kgdb into the mainline via x86 architecture updates failed, but Torvalds did [open the door a crack](<http://lwn.net/Articles/267958/>) towards merging the kgdb changes: ""I won't even consider pulling it unless it's offered as a separate tree, not mixed up with other things. At that point I can give a look."" That has spawned the kgdb-light effort, spearheaded by Ingo Molnar.

The original [hope to get it included into 2.6.25](<http://lwn.net/Articles/268589/>) has been dashed, but with Molnar rapidly iterating to address kernel hacker concerns, the amount of complaints seems to be decreasing. Molnar is up to [version 10 of the kgdb-light patchset](<https://lwn.net/Articles/270170/>) in something like three days since the first was posted. The various linux-kernel threads show a number of very hopeful developers waiting with bated breath to see if kgdb can finally make its way into the mainline. 

The light version of kgdb still has most of the capabilities of the original kgdb and any additional, possibly more intrusive, features can be added later. Molnar is clearly trying to do things the right way, with a merge of the non-intrusive kgdb functionality that can eventually be used by multiple architectures. He points out that there are already gdb remote stubs in three separate architectures in the mainline, continuing: 

So we could have done it the same way, by doing cp kernel/kgdb.c arch/x86/kernel/gdb-stub.c and merging that. Nobody could have said a _single_ word - we already have lowlevel UART code in early_printk.c that we could have reused. 

But we wanted to do it _right_ and not add an arch/x86/kernel/gdb-stub.c special hack. 

Discussions about the patches have been mostly to point out problems or areas that need cleaning up. The philosophical objections have been mostly avoided, quite possibly because Molnar has been scrupulously trying to make a "no impact" set of patches: 

this kgdb series has _obviously_ zero impact on the kernel, because it just does not touch any dangerous codepath. From this point on KGDB can evolve in small, well-controlled baby steps, as all other kernel code as well. 

To that end, the patch changes 22 files (rather than the 41 touched by the original kgdb submission), removing ""_all_ critical path impact"" and the low-level serial drivers--as Molnar points out, kgdb should not be in the driver business. In addition, the "kgdb over polled consoles" support has been reworked and cleaned up. Various hacks to get at module symbols have been removed as a better solution for that problem is needed. So far, no show stopping problems have been identified, so it really seems to come down to what Torvalds thinks; for that, we may have to wait until the 2.6.26 merge window opens in April or May.
