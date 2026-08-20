---
title: The first kpatch submission
url: https://lwn.net/Articles/597407/
date: "May 7, 2014"
category: Live patching
author: "By Jonathan Corbet May 7, 2014"
---

By **Jonathan Corbet**  
May 7, 2014

It is spring in the northern hemisphere, so a young kernel developer's thoughts naturally turn to … dynamic kernel patching. Last week saw the posting of SUSE's [kGraft](<https://lwn.net/Articles/596854/>) live-patching mechanism; shortly thereafter, developers at Red Hat came forward with their competing [kpatch](<https://lwn.net/Articles/597123/>) mechanism. The approaches taken by the two groups show some interesting similarities, but also some significant differences. 

Like kGraft, kpatch replaces entire functions within a running kernel. A kernel patch is processed to determine which functions it changes; the kpatch tools (not included with the patch, but available in [this repository](<https://github.com/dynup/kpatch>)) then use that information to create a loadable kernel module containing the new versions of the changed functions. A call to the new `kpatch_register()` function within the core kpatch code will use the ftrace function tracing mechanism to intercept calls to the old functions, redirecting control to the new versions instead. So far, it sounds a lot like kGraft, but that resemblance fades a bit once one looks at the details. 

KGraft goes through a complex dance during which both the old and new versions of a replaced function are active in the kernel; this is done in order to allow each running process to transition to the "new universe" at a (hopefully) safe time. Kpatch is rather less subtle: it starts by calling `stop_machine()` to bring all other CPUs in the system to a halt. Then, kpatch examines the stack of every process running in kernel mode to ensure that none are running in the affected function(s); should one of the patched functions be active, the patch-application process will fail. If things are OK, instead, kpatch patches out the old functions completely (or, more precisely, it leaves an ftrace handler in place that routes around the old function). There is no tracking of whether processes are in the "old" or "new" universe; instead, everybody is forced to the new universe immediately if it is possible. 

There are some downsides to this approach. `stop_machine()` is a massive sledgehammer of a tool; kernel developers prefer to avoid it if at all possible. If kernel code is running inside one of the target functions, kpatch will simply fail; kGraft, instead, will work to slowly patch the system over to the new function, one process at a time. Some functions (examples would include `schedule()`, `do_wait()`, or `irq_thread()`) are always running somewhere in the kernel, so kpatch cannot be used to apply a patch that modifies them. On a typical system, there will probably be a few dozen functions that can block a live patch in this way — a pretty small subset of the thousands of functions in the kernel. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

While kpatch, with its use of `stop_machine()`, may seem heavy-handed, there are some developers who would like to see it take an even stronger approach initially: Ingo Molnar [suggested](<https://lwn.net/Articles/597417/>) that it should use the process freezer (normally used when hibernating the system) to be absolutely sure that no processes have any running state within the kernel. That would slow live kernel patching even more, but, as he [put it](<https://lwn.net/Articles/597418/>): 

Well, if distros are moving towards live patching (and they are!), then it looks rather necessary to me that something scary as flipping out live kernel instructions with substantially different code should be as safe as possible, and only then fast. 

The hitch with this approach, as [noted](<https://lwn.net/Articles/597420/>) by kpatch developer Josh Poimboeuf, is that there are a lot of unfreezable kernel threads. Frederic Weisbecker [suggested](<https://lwn.net/Articles/597421/>) that the [kernel thread parking mechanism](<https://lwn.net/Articles/500338/>) could be used instead. Either way, Ingo thought, kernel threads that prevented live patching would be likely to be fixed in short order. There was not a consensus in the end on whether freezing or parking kernel threads was truly necessary, but opinion did appear to be leaning in the direction of being slow and safe early on, then improving performance later. 

The other question that has come up has to do with patches that change the format or interpretation of in-kernel data. KGraft tries to handle simple cases with its "universe" mechanism but, in many situations, something more complex will be required. [According to kGraft developer Jiri Kosina](<https://lwn.net/Articles/597426/>), there is a mechanism in place to use a "band-aid function" that understands both forms of a changed data structure until all processes have been converted to the new code. After that transition has been made, the code that writes the older version of the changed data structure can be patched out, though it may be necessary to retain code that reads older data structures until the next reboot. 

On the kpatch side, instead, there is currently no provision for making changes to data structures at all. The plan for the near future is to add a callback that can be packaged with a live patch; its job would be to search out and convert all affected data structures while the system is stopped and the patch is being applied. This approach has the potential to work without the need for maintaining the ability to cope with older data structures, but only if _all_ of the affected structures can be located at patching time — a tall order, in many cases. 

The good news is that few patches (of the type that one would consider for live patching) make changes to kernel data structures. As Jiri [put it](<https://lwn.net/Articles/597430/>): 

We've done some very light preparatory analysis and went through patches which would make most sense to be shipped as hot/live patches without enough time for proper downtime scheduling (i.e. CVE severity high enough (local root), etc). Most of the time, these turn out to be a one-or-few liners, mostly adding extra check, fixing bounds, etc. There were just one or two in a few years history where some extra care would be needed. 

So the question of safely handling data-related changes can likely be deferred for now while the question how to change the code in a running kernel is answered. There have already been [suggestions](<http://lists.linuxfoundation.org/pipermail/ksummit-discuss/2014-May/000018.html>) that this topic should be discussed at the 2014 Kernel Summit in August. It is entirely possible, though, that the developers involved will find a way to combine their approaches and get something merged before then. There is no real disagreement over the end goal, after all; it's just a matter of finding the best approach for the implementation of that goal.
