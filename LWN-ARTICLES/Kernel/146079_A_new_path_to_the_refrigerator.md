---
title: A new path to the refrigerator
url: https://lwn.net/Articles/146079/
date: "August 3, 2005"
category: Software suspend
author: 
---

One of the trickier parts of the software suspend subsystem is the "refrigerator," the code which puts all processes on hold so that the system can be suspended in a quiet state. [Last week](<http://lwn.net/Articles/144453/>), this page looked at some issues which come up in choosing which processes to freeze and when to freeze them. Another area of work, however, is the mechanism by which the freezing actually happens. 

The in-kernel software suspend code puts processes on hold with the following steps: 

  * The process flags (stored in the `flags` field of the `task_struct` structure) gets the `PF_FREEZE` bit set. 

  * A signal is delivered to the process, causing it to execute briefly. 

  * Eventually the process notices the `PF_FREEZE` flag and calls `refrigerator()`. That call replaces `PF_FREEZE` with `PF_FROZEN` and puts the process into an unrunnable state (`TASK_UNINTERRUPTIBLE`). 

This mechanism does work, but it has a couple of problems. The `PF_*` flags require some support in the scheduler, which would be nice to avoid. The real issue, though, is that accessing another process's flags requires locking to avoid race conditions. Adding that sort of locking to the software suspend code, however, is hard to do without risking deadlocks. So the suspend code simply sets the `PF_FREEZE` flag without locking and hopes for the best; this is one of the reasons why software suspend has never really been supported on SMP systems. 

Christoph Lameter has posted [a set of patches](<https://lwn.net/Articles/145417/>) aimed at fixing these issues. With his patch, the `PF_FREEZE` and `PF_FROZEN` flags go away. Instead, `struct task_struct` gets a new field called `todo`. This field is a `notifier_block` pointer; whenever any part of the kernel wants a particular process to run a function in its own context, the kernel can put a notifier request onto `todo`. At various places in the kernel, the `todo` list is checked, and any notifier requests which have been put there are executed. 

With this mechanism, there is no need for any special process flags. The suspend code simply adds a `todo` item for each process asking it to freeze itself. It is still necessary to deliver a signal to each process to force it to run in the kernel; otherwise, processes waiting on I/O (or which never call out of user space) would not execute the notifier. The actual "frozen" state is implemented with a `completion` in Christoph's patch, meaning that unfreezing everybody is a simple matter of a call to `complete_all()`. 

Christoph thinks that the `todo` mechanism may be useful beyond software suspend. A number of places in the kernel have to make changes which are best run in the context of a specific process; the code to make those changes happen can, at times, be a little ugly. The `todo` list is a straightforward way of running code directly in the context of interest, potentially simplifying the kernel in a few places. The patch has not made it into -mm as of this writing, but there does not appear to be any great obstacle to its inclusion there.
