---
title: The interaction between memory reclaim and RCU
url: https://lwn.net/Articles/974487/
date: "May 22, 2024"
category: "Memory management-Scalability; Read-copy-update"
author: "By Jonathan Corbet May 22, 2024 LSFMM+BPF"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
May 22, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

The [2024 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>) was a development conference, where discussion was prioritized and presentations with a lot of slides were discouraged. Paul McKenney seemingly flouted this convention in a joint session of the storage, filesystem, and memory-management tracks where he presented about 50 slides — in five minutes, twice. The subject was the use of the read-copy-update (RCU) mechanism in the memory-reclaim process, and whether changes to RCU would be needed for that purpose. 

Readers who are unfamiliar with RCU may want to have a look at [this article](<https://lwn.net/Articles/262464/>) for a refresher. 

After the slide deluge (for which it was not possible to take effective notes without severe keyboard damage), McKenney got to his real point: before making complicated changes to the RCU subsystem (which does not lack complexity already), a real problem with the current code will need to be demonstrated. The concern seems to be that RCU is simply too slow in getting around to freeing memory, causing the system to go into an out-of-memory state. What can be done about that? 

[![\[Paul McKenney\]](https://static.lwn.net/images/conf/2024/lsfmm/PaulMcKenney-sm.png)](<https://lwn.net/Articles/974490/>) The question of whether RCU can keep up with the work presented to it is, he said, dependent on the workload. There can be a few reasons why it would fail; perhaps the grace periods (the delay before RCU callbacks can be run) are too infrequent, or there may be readers holding the RCU lock for too long. Kent Overstreet tried to give some background for the current topic, which is a spin-off from [a lengthy pre-conference discussion](<https://lwn.net/ml/linux-mm/Zdkxfspq3urnrM6I@bombadil.infradead.org/>) on buffered-I/O performance. The buffered read path is fast now, he said, but it can be made faster by using RCU. But that means using RCU to free page-cache pages, which is a critical cleanup path in the kernel. 

McKenney suggested that, as an experiment, developers could try just forcing page-free operations through RCU for no particular reason, just to see if anything breaks. Josef Bacik said that, while there are places where RCU can be improved, this use case is pushing for a solution to a problem that is not RCU's fault. Writeback from page-cache pages can take a long time; page reclaim is an unpredictable process in general. 

Overstreet agreed that reclaim is a hard problem, and that a lot of different developers have responsibility for parts of it. He is a filesystem developer who finds himself having to solve reclaim problems, but the kernel lacks the sort of introspection that would help him to see where the problems are. Thus, he said, there is a need for a wider discussion about the interactions around the reclaim problem. 

James Bottomley asked whether it was appropriate to use RCU in this way; perhaps there is a need to invent a new mechanism instead? McKenney answered that, instead, developers could use a different flavor of RCU, such as [sleepable RCU](<https://lwn.net/Articles/202847/>). Steven Rostedt asked whether a new RCU flavor aimed at the reclaim problem is needed, but McKenney said he did not think that was the case. 

It was this far into the session before Dave Chinner got up to ask what the problem to be solved was. The short answer is "making the buffer cache faster". Matthew Wilcox said that taking folio references for small reads is simply too expensive; RCU can be used instead to keep pages around while data is copied from them without the need to take a reference. McKenney suggested that perhaps [hazard pointers](<https://en.wikipedia.org/wiki/Hazard_pointer>) could be used for this purpose. That would allow the immediate freeing of any object that is not currently referenced; RCU, instead, must wait for all readers to complete their work. 

Bottomley said that the reference-count problem comes down to the cost of converting cache lines to exclusive access. If there is not actually a lot of contention for those reference counts, perhaps a different solution is called for. Overstreet answered that, even in the no-contention case, the reference-counting overhead is a problem; Wilcox suggested that Bottomley was underestimating the number of places in the kernel that take references. 

McKenney tried to direct the conversation toward an understanding of the performance problem; Overstreet answered that better numbers are needed. He would like to be able to track just how much memory is waiting in the RCU system to be freed. McKenney answered that, while [`kvfree_call_rcu()`](<https://elixir.bootlin.com/linux/v6.9.1/source/kernel/rcu/tree.c#L3405>) is aware of the size of the memory block it has been asked to free, it is used infrequently. Most memory is freed using [`call_rcu()`](<https://elixir.bootlin.com/linux/v6.9.1/source/kernel/rcu/tree.c#L2792>), and that function has no idea of how much memory it will eventually free (or whether it is freeing memory at all). There is also no per-subsystem accounting in RCU. Hannes Reinecke said that he would like to see subsystem-level accounting, along with the ability to force a grace period for a specific subsystem. The problem there, as somebody pointed out, is that the ability to free a specific range of memory may depend on other subsystems, and there is no way to know for sure. 

Chinner said that this is a problem of tracking objects in flight. It is possible to count slab objects, since they know which slab they belong to and their size; it's just a matter of adding the tracking. Calls to [`kfree_rcu()`](<https://elixir.bootlin.com/linux/v6.9.1/source/include/linux/rcupdate.h#L966>) could recognize slab objects and account for them. McKenney said that he would like to see `kfree_rcu()` merged into the slab allocator; slab maintainer Vlastimil Babka said that he had plans to do exactly that. Now that the SLOB allocator has been removed, he said, kernel code can pass any memory pointer to `kfree()` (and thus `kfree_rcu()`) and the right thing will happen. 

As this somewhat inconclusive session came to a close, McKenney said that there were two problems to be solved. If the system is loaded with memory demands, how are those to be accounted for? And, for memory freed with `call_rcu()`, more information will need to be provided somehow. Overstreet got in the last word by saying that, if a kernel subsystem is using `call_rcu()`, the duty of performing the accounting is also there. `kfree_rcu()` should be used instead whenever possible.
