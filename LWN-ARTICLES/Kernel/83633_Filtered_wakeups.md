---
title: Filtered wakeups
url: https://lwn.net/Articles/83633/
date: "May 5, 2004"
category: "Filtered wakeups; Scheduler-Filtered wakeups; Wait queues"
author: 
---

Kernel code often finds itself having to wait for a particular physical page; if, for example, a page is currently under I/O, prospective users must wait until that operation has completed. In the early days of 2.4 (and before), the `struct page` structure (which the kernel uses to track physical memory) contained a wait queue head for this purpose. This technique worked, but adding a wait queue for every page in the system was not a particularly efficient use of memory. At any given time, only a tiny percentage of those wait queues are actually in use. 

To recover some of the memory used by wait queues, the kernel developers added the concept of hashed wait queues. The per-page queues were replaced with a much smaller number of shared queues; when a thread needs to wait on a particular page, it hashes the page address to pick the appropriate queue. When the page becomes available, all processes waiting on that queue will be awakened. The use of this technique has since been extended to other parts of the kernel as well. 

Hashed wait queues have achieved the desired space savings, but, as it turns out, at a certain computational cost. William Lee Irwin [did some research](<https://lwn.net/Articles/83635/>), and found that hash queue collisions are fairly common. So, when a wakeup is performed on one of the hashed wait queues, it is likely that unrelated processes are being awakened. Each of those processes must run, determine that the event they are waiting for has not yet occurred, and go back to sleep. This variant on the "thundering herd" problem can hurt performance. 

One possible solution to this problem would be to expand the number of wait queues to make collisions less likely. That approach is simple, but it also would bring back the original problem by expanding the amount of memory dedicated to wait queues. So William came up with another approach, which he calls "filtered wakeups." 

The idea behind a filtered wakeup is fairly simple. When a process goes to sleep on a (shared) filtered wait queue, it provides a "key" value, which will typically be the address of the resource being waited for. The wakeup call is made with a key value as well; as the wait queue is traversed, only the processes waiting for the given key are awakened. 

[The patch](<https://lwn.net/Articles/83507/>) which implements filtered waits is fairly simple, and includes an example of their use. It creates a new `filtered_wait_queue` structure: 

```
struct filtered_wait_queue {
    		void *key;
    		wait_queue_t wait;
    	};
```

A process which is about to go into a filtered wait will use code which looks something like the following to create an use a filtered queue entry: 

```
DEFINE_FILTERED_WAIT(wait, key);
    
    	do {
    		prepare_to_wait(queue, &wait.wait, TASK_INTERRUPTIBLE);
    		if (not_ready_yet(key))
    			schedule();
    	} while(not_ready_yet(key));
    	finish_wait_(queue, &wait.wait);
```

Awakening a process in this sort of sleep is a simple matter of calling: 

```
void wake_up_filtered(wait_queue_head_t *queue, void *key);
```

William [claims](<https://lwn.net/Articles/83506/>) some significant performance improvements from his changes, including large reductions in CPU usage and a near tripling of the peak I/O rates in some situations.
