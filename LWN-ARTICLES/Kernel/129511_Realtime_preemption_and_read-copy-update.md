---
title: "Realtime preemption and read-copy-update"
url: https://lwn.net/Articles/129511/
date: "March 30, 2005"
category: "Latency; Preemption; Read-copy-update; Realtime"
author: 
---

Ingo Molnar's massive realtime preemption patch is an attempt to bring near-realtime response to the stock Linux kernel. It works by making almost everything in the kernel preemptible. Spinlocks turn into preemptible mutexes; interrupt handlers get moved into preemptible kernel threads, etc. The result is a major change in how the scheduling of kernel code is done and quick response to external events. This work has been quieter in recent times, but it has not stalled by any means. 

When LWN last [looked at the realtime preemption patch](<https://lwn.net/Articles/107269/>), one of the remaining rough spots was its interaction with the read-copy-update (RCU) mechanism. RCU, remember, encapsulates a conceptually simple (though a bit more gnarly in the implementation) technique. A resource of interest (a routing table entry, say) is referenced by a pointer. When that resource must be changed, a copy is made and the changes are done there; the pointer is then directed at the new copy. At some future, safe time, the old version can be freed. Linux RCU works by requiring that all accesses to RCU-protected data structures be atomic; with that constraint, a "safe time" can be defined as "after every processor on the system has scheduled." Since scheduling while holding a reference to an RCU-protected structure is against the rules, any such structure which was made inaccessible before all processors schedule cannot be referenced by any processor afterward. 

Since accesses to RCU-protected structures must be atomic, the RCU locking function (`rcu_read_lock()`) disables preemption. But disabling preemption is exactly what the realtime preemption patch is trying to get away from, so something had to give. Ingo had solved this problem by requiring that all RCU users identify an explicit lock which protects the structures in question, and modifying the RCU locking functions to take that lock as a parameter. This approach was never optimal. It caused the creation of a whole new family of new RCU functions to cope with every type of lock that might be used, and, simultaneously, decreased the flexibility of the RCU read locking mechanism. And, to a great extent, it simply replaced RCU with more traditional locking which, while it works, does not have the scalability advantages which were the motivation for RCU in the first place. 

The RCU issue was clearly [on Ingo's mind](<https://lwn.net/Articles/129563/>): 

If PREEMPT_RT is merged into the upstream kernel then it will (at least initially) be at a status similar to NOMMU: it will be tolerated as long as it causes no 'drag' on the main code. The RCU API variants i introduced clearly violated this requirement, and were my #1 worry wrt. upstream mergability. 

So Ingo was pleased when RCU creator Paul McKenney [proposed some approaches](<https://lwn.net/Articles/128228/>) for making RCU and realtime preemption work together. Paul's message goes through a series of increasingly complex solutions, and is worth reading in its own right. The core idea, however, is that, in a fully preemptible world, RCU cannot depend on atomic access to data structures, and thus cannot use the "all processors have scheduled" heuristic to know that the time has come to execute a given set of RCU cleanup functions. So the tracking of code executing within RCU critical sections must be made more explicit. Paul's solutions used a reader/writer lock for that purpose, but the approach taken in Ingo's [latest realtime preemption patch](<https://lwn.net/Articles/129211/>) is a little different. 

The code executed to go into an RCU-protected section now looks like this (when configured for realtime preemption): 

```
void rcu_read_lock(void)
        {
    	if (current->rcu_read_lock_nesting++ == 0) {
    		current->rcu_data = &get_cpu_var(rcu_data);
    		atomic_inc(&current->rcu_data->active_readers);
    		smp_mb__after_atomic_inc();
    		put_cpu_var(rcu_data);
    	}
        }
```

The idea is simple: a per-CPU count of processes in RCU critical sections is kept. When a process goes into a critical section, a pointer to the current CPU's counter is stored with the task information, so that the right counter will be decremented later on. There is also a per-process variable which keeps track of RCU section nesting. No further work needs to be done before the process can access the protected structure; in particular, no locks are acquired. 

When the process exits the critical section, the process is reversed: the nesting count is decremented. When that count goes to zero, the per-CPU count is decremented as well. If the per-CPU count drops to zero, then that processor is deemed to have "quiesced," with no processes running within RCU critical sections. Once all CPUs have quiesced in this way (as tracked by a bitmask of processors in the system), all RCU cleanup functions queued before their respective processors quiesced can be called. 

This scheme restores the core RCU functionality, allowing lock-free access to fast-path data structures. It also retains the current RCU API, with the result that the realtime preemption patch becomes significantly less intrusive. It is not a perfect implementation, however. It requires that each CPU regularly find itself with no processes executing within RCU critical sections. Since these sections are now preemptible, the "quiet" times could be quite far apart on heavily-loaded systems. While the system is waiting for a processor to quiesce, the RCU callback structures for the cleanup functions will continue to accumulate, to the point that quite a bit of memory could be used before the cleanup actually happens. For the realtime case, this tradeoff is acceptable: latency, not memory use, is the most important factor. Since the existing RCU algorithm is used when realtime preemption is not configured in, everybody should be happy. In practice, further work may be required; in particular, it may be necessary to find a way to force RCU cleanup when the system gets low on memory. Meanwhile, however, the realtime preemption patch appears to have gotten past one more major hurdle on its way toward possible inclusion into the mainline.
