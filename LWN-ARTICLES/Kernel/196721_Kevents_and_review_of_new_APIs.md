---
title: Kevents and review of new APIs
url: https://lwn.net/Articles/196721/
date: "August 23, 2006"
category: "Development model-User-space ABI; Documentation; Kevent"
author: 
---

The proposed kevent interface has been covered here before - see [this article](<http://lwn.net/Articles/193691/>) and [this one too](<http://lwn.net/Articles/172844/>). Kevents appear to have gained significant momentum over the last few weeks, to the point that inclusion in 2.6.19 is not entirely out of the question. Most developers who have reviewed the code seem to like the core idea (a unified interface for applications to get information on all events of interest) and the implementation within the kernel. Only now, however, is significant attention being paid to the user-space API which comes with kevents. But the definition of that API is of crucial importance. This article will look at it from two perspectives - first technical, then political. 

The discussion of the proposed API has been hampered somewhat by the lack of associated documentation - and the fact that said API is still changing quickly. In an attempt to pull together some of the available information, Stephen Hemminger has put up [a page at OSDL](<http://linux-net.osdl.org/index.php/Kevent>) describing the system call API. That page misses one important aspect of kevents, however: the ability to receive events via a shared memory interface. In an attempt to fill that gap, we'll look at the August 23 version of the memory-mapped kevent API. 

One of the goals behind kevents is to make the processing of events as fast as possible - the idea being that a high-performance network server (say) can work through vast numbers of events per second without appreciable system overhead. One way to achieve this is to avoid system calls altogether whenever possible. That is why there is interest in mapping kevents directly into user space; this approach will allow the application to consume them without calling into the kernel for each one. 

To use the mmap interface, the application obtains a kevent file descriptor, as usual. A simple call to `mmap()` will then create the shared buffer for kevent communication. The size of this buffer is currently determined by an in-kernel parameter - the maximum number of kevents which will be stored there. Presumably there will eventually be a `KEVENT_MMAP_PAGES` macro (or some such) to free the application from trying to figure out how many pages it should map, but that is not yet provided. 

The resulting memory array is treated as a big circular buffer by the kernel. There is a single index only, however - where the next event will be written by the kernel. In other words, the kernel has no way to know which events have been consumed by the application; if that application falls too far behind, the kernel will begin to overwrite unprocessed events. For this reason, perhaps, the buffer is made relatively large - 4096 events fit there in the current version of the patch. 

The events stored in the buffer are not the same `ukevent` structures used by the system call interface. There is, instead, a shortened version in the form of `struct mukevent`: 

```
struct kevent_id
        {
    	union {
    	    __u32 raw[2];
    	    __u64 raw_u64 __attribute__((aligned(8)));
    	};
        };
    
        struct mukevent
        {
    	struct kevent_id	id;
    	__u32			ret_flags;
        };
```

The `id` field contains some information about what happened: the relevant file descriptor, for example. The actual event code itself is not present, however. 

The event ring is not quite a pure circular buffer. It is formatted with a four-byte field at the beginning of each page, followed by as many `mukevent` structures as will fit within the page. The four-byte field in the first page contains the current event index - where the kernel will write the next event. The application will, presumably, keep track of the last event it read from the buffer, moving that counter forward until it catches up with the kernel. The application must take care, however, to notice every time it crosses a page boundary so it can skip the count field. 

Since there is no way to inform the kernel that events have been consumed from the memory-mapped ring, and since the full event information is not available via that ring, the application must still call into the kernel for events. Otherwise, if nothing else, they will accumulate there until they reach their maximum allowed number. So the advantage of the memory-mapped approach will be hard to obtain with the current code. As was noted above, however, this API is very young. One assumes that these little problems will be ironed out in the near future. 

Meanwhile, kevents have created a separate discussion on how new APIs go into the kernel. One Nicholas Miell [requested](<https://lwn.net/Articles/196727/>) that some documentation for this interface be written: 

Is any of this documented anywhere? I'd think that any new userspace interfaces should have man pages explaining their use and some example code before getting merged into the kernel to shake out any interface problems. 

The [response](<https://lwn.net/Articles/196728/>) he got was ""Get real"". Others suggested that, if Mr. Miell really wanted documentation, he could sit down and write it himself. It must be said that, through the discussion, Mr. Miell has comported himself in a way which is highly unlikely to inspire cooperation from anybody. He seems to carry a certain contempt for the interface, the process, and the people involved in it. 

But it must also be said that he has a point. The creation of user-space APIs differs from how most kernel code is written. Much is made of the evolutionary nature of the kernel itself - things continually evolve as better solutions to problems are found. User-space interfaces, however, cannot evolve - once they are shipped as part of a mainline kernel, they are set in stone and must be maintained forever. They must be right from the outset. So it is not unreasonable to expect that the level of review for new user-space APIs would be higher, and that documentation of proposed APIs, which can be expected to help the review process, should be provided. It is true, however, that the original developer is not always the best person to provide that documentation. 

One question which has been raised about this interface has to do with its similarity to the FreeBSD [kqueue](<http://www.freebsd.org/cgi/man.cgi?query=kqueue>) mechanism. The intent of the interface is the same, but no attempt to emulate the kqueue API has been made. Andrew Morton [has said](<https://lwn.net/Articles/196729/>): 

I mean, if there's nothing wrong with kqueue then let's minimise app developer pain and copy it exactly. If there _is_ something wrong with kqueue then let us identify those weaknesses and then diverge. Doing something which looks the same and works the same and does the same thing but has a different API doesn't benefit anyone. 

There are, evidently, real reasons for not replicating the kqueue interface, but those reasons have not, yet, been made clear. 

Kevents will, it is hoped, be a major improvement for people writing applications for Linux. This new API should bring together all information of interest into a single place, provide significant performance benefits, and ease porting of applications from other operating systems. But, if this API is going to meet the high expectations being placed on it, it will require a high level of review from a number of interested parties. That review is now starting to happen, so expect this API to remain in flux for some time yet.
