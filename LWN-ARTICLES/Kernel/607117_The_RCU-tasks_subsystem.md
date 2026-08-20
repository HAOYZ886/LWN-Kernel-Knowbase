---
title: "The RCU-tasks subsystem"
url: https://lwn.net/Articles/607117/
date: "July 30, 2014"
category: "Read-copy-update"
author: "By Jonathan Corbet July 30, 2014"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
July 30, 2014

The read-copy-update (RCU) mechanism is charged with keeping old versions of data structures around until it knows that no CPU can hold a reference to them; once that happens, the structures can be freed. Recently, though, a potential RCU user came forward with a request for something different: would it be possible to defer the destruction of an old data structure until it is known that no _process_ holds a reference to it? The answer would appear to be "yes," as demonstrated by the [RCU-tasks subsystem](<https://lwn.net/Articles/606959/>) recently posted by Paul McKenney. 

Normal RCU works on data structures that are accessed via a pointer. When an RCU-protected structure must change, the code that maintains that structure starts by making a copy. The changes are made to the copy, then the relevant pointer is changed to point to that new copy. At this point, the old version is inaccessible, but there may be code running that obtained a pointer to it before the change was made. So the old structure cannot yet be freed. Instead, RCU waits until every CPU in the system goes through a context switch (or sits idle). Since the rules for RCU say that references to data structures can only be held in atomic context, the "every CPU has context switched" condition guarantees that no references to an old data structure can be held. 

It seems that the rules for the trampolines used by the tracing code are different, though, in that a process can be preempted while still holding a reference to (i.e. running within) an old version. Given that, normal RCU will not work for the management of these structures, meaning that some other, slower locking mechanism must be used. Using an RCU-like mechanism would require that the rules be changed somewhat. 

In the normal RCU case, only one process can hold a reference to a protected structure on any given CPU; as a result, RCU focuses on figuring out when no CPU can hold a reference to a given data structure. In this case, there might be multiple processes on each CPU with a reference to the protected data structure, so the focus has to shift. Thus, RCU-tasks is a mechanism designed to figure out when no processes (rather than no processors) can hold such a reference. 

With this interface, code that has replaced a protected data structure will arrange for the disposal of the old version with a call to: 

```
void call_rcu_tasks(struct head *rhp, void (*func)(struct rcu_head *rhp));
```

Once the appropriate "grace period" has passed, `func()` will be called with the given `rhp` to free the structure. For users of RCU-tasks, that is pretty much the entire API. Unlike ordinary RCU, RCU-tasks has no equivalent to `rcu_read_lock()` for access to protected data structures. 

Ordinary RCU has, over the years, acquired a great deal of complexity in order to maximize the scalability of the subsystem. RCU-tasks, instead, is refreshingly simple, at least in its initial implementation. There is a single linked list of `rcu_head` structures that have been passed to `call_rcu_tasks()` but that have not yet been acted upon. The patch set adds a new kernel thread charged with managing that list. Once every second, it wakes up to see if any new entries have been added to the list (a subsequent patch replaces the poll with a wait queue). If so, the entire list is moved to a separate list, and the wait for a new grace period to pass begins. 

That wait starts by creating a separate list of every runnable process in the system; tasks that are not runnable cannot, by the rules, hold a reference to data structures protected by RCU-tasks, and, thus, need not be considered. For each runnable task, a special "`rcu_tasks_holdout`" flag is set in the task structure. Hooks have been placed in the scheduler to clear that flag whenever the task voluntarily gives up the CPU or returns to user space. The RCU-tasks kernel thread goes into a separate loop, waking up every tenth of a second, to work through the list of "holdout" tasks; any that have had their flag reset are removed from the list. Once the list is empty, the destructor callbacks can be called and the cycle can start anew. 

The code gets somewhat more complex as the patch series goes on. The addition of testing infrastructure and stall detection adds somewhat to its footprint. The biggest addition, though, is the addition of handling of tasks that exit while they are on the holdout list. Clearly, checking for the "holdout" flag in a task structure that may no longer exist is a bad idea, so this case does need to be properly handled. Doing so involves adding a new type of lock-protected doubly linked list and a bunch of management code; it is the biggest part of the entire patch set. 

Thus far, we have not yet seen patches to make other code actually use this new facility. Most of the comments on this patch set have come from Peter Zijlstra, who is [concerned](<https://lwn.net/Articles/607138/>) about the overhead of polling and the lack of accounting of that overhead. So there are a few questions yet to be answered. While RCU-tasks may well prove to be a useful addition to the RCU API, nobody is expecting to see it in the 3.17 merge window.
