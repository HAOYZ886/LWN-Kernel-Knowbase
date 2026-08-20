---
title: Clusters and distributed lock management
url: https://lwn.net/Articles/136308/
date: "May 18, 2005"
category: Clusters; Distributed lock management
author: 
---

The creation of tightly-connected clusters requires a great deal of supporting infrastructure. One of the necessary pieces is a lock manager - a system which can arbitrate access to resources which are shared across the cluster. The lock manager provides functions similar to those found in the locking calls on a single-user system - it can give a process read-only or write access to parts of files. The lock management task is complicated by the cluster environment, though; a lock manager must operate correctly regardless of network latencies, cope with the addition and removal of nodes, recover from the failure of nodes which hold locks, etc. It is a non-trivial problem, and Linux does not currently have a working, distributed lock manager in the mainline kernel. 

David Teigland (of Red Hat) recently [posted](<https://lwn.net/Articles/136066/>) a set of distributed lock manager patches (called "dlm"), with a request for inclusion into the mainline. This code, which was originally developed at Sistina, is said to be influenced primarily by the venerable VMS lock manager. An initial look at the code confirms this statement: callbacks are called "ASTs" (asynchronous system traps, in VMS-speak), and the core locking call is an eleven-parameter monster: 

```
int dlm_lock(dlm_lockspace_t *lockspace,
    	int mode,
    	struct dlm_lksb *lksb,
    	uint32_t flags,
    	void *name,
    	unsigned int namelen,
    	uint32_t parent_lkid,
    	void (*lockast) (void *astarg),
    	void *astarg,
    	void (*bast) (void *astarg, int mode),
    	struct dlm_range *range);
```

Most of the discussion has not been concerned with the technical issues, however. There are some disagreements over issues like how nodes should be identified, but most of the developers who are interested in this area seem to think that this implementation is at least a reasonable starting point. The harder issue is figuring out just how a general infrastructure for cluster support can be created for the Linux kernel. At least two other projects have their own distributed lock managers and are likely to want to be a part of this discussion; an Oracle developer recently [described](<https://lwn.net/Articles/136312/>) the posting of dlm as "a preemptive strike." Lock management is a function needed by most tightly-coupled clustering and clustered filesystem projects; wouldn't it be nice if they could all use the same implementation? 

The fact is that the clustering community still needs to work these issues out; Andrew Morton [doesn't want to have to make these decisions](<https://lwn.net/Articles/136315/>) for them: 

Not only do I not know whether this stuff should be merged: I don't even know how to find that out. Unless I'm prepared to become a full-on cluster/dlm person, which isn't looking likely. 

The usual fallback is to identify all the stakeholders and get them to say "yes Andrew, this code is cool and we can use it", but I don't think the clustering teams have sufficent act-togetherness to be able to do that. 

Clustering will be discussed at the kernel summit in July. A month prior to that, there will also be [a clustering workshop](<http://lwn.net/Articles/136490/>) held in Germany. In the hopes that these two events will help bring some clarity to this issue, Andrew has said that he will hold off on any decisions for now.
