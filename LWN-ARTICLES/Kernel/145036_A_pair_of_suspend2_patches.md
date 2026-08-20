---
title: A pair of suspend2 patches
url: https://lwn.net/Articles/145036/
date: "July 26, 2005"
category: Software suspend; Workqueues
author: 
---

One of the outcomes from the power management summit held on July 17 was a decision to move toward merging Nigel Cunningham's suspend2 patches - at least, those which appear to make sense to the wider community. Suspend2 is an out-of-tree implementation of the suspend-to-disk and suspend-to-ram features which are so nice to have on laptop systems. The suspend2 implementation offers a number of features not found in the mainline version, including nice displays, progress bars, interruptible operation, and, it is said, greater reliability. Getting the better parts of suspend2 into the mainline seems like a clearly desirable goal. Since the summit, Nigel has posted a pair of patches which provide some clues as to what is coming, and how it will be received. 

A big part of the software suspend problem is getting the system into a quiescent state before putting it on ice. To that end, processes are put into the "refrigerator," a special sort of suspended animation. When suspend time comes around, every process on the system is sent a special signal telling it that refrigeration time has come; each process, once it gets to a good stopping point, checks itself into the fridge and does not run again until after the system has been resumed. 

The problem that this scheme runs into is that some processes are dependent on others. If a process which, for example, is involved with getting data written to disk is refrigerated too early in the process, it may never be possible to get the system to a state where it can be suspended. So the software suspend patches try to figure out which processes must be allowed to continue running while the system is being quiesced. It has always been a bit of a hit-and-miss business. The current suspend2 patches try to clean up that process a bit. 

Many of the processes which should not be refrigerated are associated with various driver workqueues. So the mainline suspend code marks every workqueue process with the special `PF_NOFREEZE` flag, keeping it out of the clutches of the refrigerator. But most of those processes can be refrigerated just fine with no ill effect, and they should be. Having unneeded processes running when the system is trying to suspend itself can only serve to destabilize the entire situation. 

Previous versions of the suspend2 patches changed the workqueue API so that every creator of a workqueue had to explicitly state whether it should be refrigerated or not. That approach worked, but it broke every `create_workqueue()` call. The [current patch](<http://lwn.net/Articles/144504/>), instead, leaves the existing calls alone, but extends the API with a couple of new calls: 

```
struct workqueue_struct *create_nofreeze_workqueue(const char *name);
        struct workqueue_struct *create_nofreeze_singlethread_workqueue(const char *name);
```

As an aside, one notes that kernel namespace is starting to acquire some very long function names. One might almost wish for the good old days, when only the first six characters of a function name were used. 

Seriously, however, these functions show how refrigeration is now handled with workqueues. By default, worker tasks associated with workqueues will be put on ice when the system is suspended. Anybody wishing to create a workqueue which does _not_ behave that way must call one of the new functions. 

This change has been propagated down to the generic kernel threads layer, which also picked up a new function: 

```
struct task_struct *kthread_nofreeze_create(int (fn)(void *data),
                                                    void *data,
    						const char *namefmt, ...);
```

This patch seems likely to be merged with, at most, minor tweaks. Nigel's [second patch](<http://lwn.net/Articles/144502/>), however, got a somewhat less friendly reception. It creates a new process flag called `PF_SYNCTHREAD`. Any process which is actively trying to flush data to disk is marked with this flag; the end result is that it will be passed over by the refrigerator during the early part of the suspend process. In this way, processes which are creating dirty pages can be put on hold prior to those which are trying to clean those pages up. This patch is not popular, however; it has been criticized for being overly intrusive when simply flushing all pages to disk prior to beginning the suspend process would do the trick. So, unless things change, this patch will not go in. 

In any case, these patches are just preparatory work for a larger event: the merging of a new refrigerator implementation. That code has not (recently) been posted; stay tuned.
