---
title: "A lockless ring-buffer"
url: https://lwn.net/Articles/340400/
date: "July 8, 2009"
category: Tracing
author: "By Jake Edge July 8, 2009"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
July 8, 2009

One of the outcomes from last year's Kernel Summit and Linux Plumbers Conference was a [plan to create a low-level ring-buffer implementation](<http://lwn.net/Articles/300992/>) that could be shared among the various kernel and user-space tracing solutions available for Linux. One implementation of the common ring-buffer was released as part of 2.6.28, but it was somewhat lock-heavy, which impacted its performance. Recently, Steven Rostedt has [proposed](<http://lwn.net/Articles/336961/>) a lockless ring-buffer algorithm, which would eliminate locking on writes, which is the fast path for tracing. 

As tracing information is gathered in the kernel, it needs to be stored somewhere very quickly, so that the impact on the timing of the events observed--and system performance overall--is fairly minimal. Reading the data is done from user space, though, so it is generally not performance-sensitive. The current ring-buffer implementation creates a circular, doubly-linked list of pages, along with a head and tail pointer, so writes are done at the tail, while reads are done from the head. 

If the ring-buffer gets full, or nearly so, there is the potential for writers to overwrite data in the head page, which could corrupt data that is being read. For this reason, there is a separate reader page, which has been removed from the list entirely, that reader processes can use without being concerned about corruption from writers. But, having that separate page requires that there be a bit of a dance whenever the reader is done with the page and needs a new one. The reader page must be placed back into the list somewhere after the tail, while the current head page needs to be removed as the new reader page, and the head page must be pushed forward. That requires locking. 

The diagram below, from Rostedt's [`ring-buffer-design.txt` document](<http://lwn.net/Articles/340443/>), gives an idea of how the ring-buffer would look. Observant readers will note the `H` pointer, which is the `HEADER`-flagged pointer described below. 

```
reader page
    		  |
    		  v
    		 +---+
    		 |   |------+
    		 +---+      |
    			    |
    			    v
    	+---+    +---+    +---+    +---+
        <---|   |--->|   |-H->|   |--->|   |--->
        --->|   |<---|   |<---|   |<---|   |<---
    	+---+    +---+    +---+    +---+
```

Writers can be interrupted by other writers, so long as the interrupting writer completes its write before the interrupted writer can continue. This is in keeping with the way interrupts stack, and it is important that it be enforced for the integrity of the ring-buffer structure. When a write is initiated, space is reserved after the tail pointer to hold the event. This moves the tail pointer, so another pointer, called the commit pointer, is needed to track the latest complete write. 

In nearly empty ring-buffers, it is possible for the reader page to also be the commit and tail pages. While the reader page has been removed from the ring-buffer, its `next` pointer still leads to the next ring-buffer entry. Once enough writes are done, the commit and tail pointers will simply follow the `next` pointer as they normally do. 

In order to remove the locking for writers, which currently need to use locks to synchronize updates of the head, tail, and commit pointers, Rostedt leverages the `cmpxchg()` atomic operation available on some architectures. It works as follows: 

```
R = cmpxchg(A, C, B) 
            - Assign A = B if A == C
            - Return A at the time of the call, unconditionally
```

The success of the exchange can be determined by checking whether `R` is equal to `C`, if so, the exchange was done. 

The algorithm requires that the pointers to the linked-list structures be 4-byte aligned so that it can reserve the bottom 2 bits for flags. The two flags are: 

  * `HEADER` \- the pointer is to the current head page 

  * `UPDATE` \- the pointer is to a page that is currently being written and either is, or is about to be, the head page 
These flags, along with the `cmpxchg()` operation, are what allow lockless writing to the buffer. 

When the reader page has been exhausted, the current head page needs to be detached from the ring-buffer as the new reader page. By using the `HEADER` flag on the `next` pointer that points to the head page, writers can keep readers from interfering without taking a lock. When trying to change the `next` pointer as part of the swapping process, readers use `cmpxchg()` to require that the `HEADER` flag be present. Writers can prevent that from happening by setting the flag to `UPDATE` or clearing the flags entirely. When the reader's `cmpxchg()` fails, it means that writers have changed the state of the ring, so the reader must look for a new head page and start the process all over. 

When writers change to a new tail page, as they fill the buffer, they check the `next` pointer of the new page for the `HEADER` flag. If it is present, it is changed to `UPDATE`. That indicates that the page is volatile, as writers are currently using it, and will cause the `cmpxchg()` of a reader to fail, should it try to detach the head page. This is an indication that the buffer is close to full, only one page (i.e. the new tail page) remains for storing events. 

The ring-buffer can operate in two modes, overwrite (aka "flight recorder") mode, where new events overwrite older events when the buffer fills up, or producer/consumer mode where writing to a full buffer causes the write to fail. In producer/consumer mode, the head page only changes at the behest of a reader, but in overwrite mode, once the tail page reaches the head, the head must be pushed forward one page, which is why the `UPDATE` flag must be used. 

The basic function of the algorithm is relatively straightforward--if a bit head-exploding--but there are number of more complex scenarios to consider. One is the possibility that nested writes cause the buffer to fill, such that the tail page reaches the commit page. There is no choice but to drop writes at that point, but it is possible that the commit page is on the reader page (as shown below). Naively pushing the head page forward, past the entry that the reader page points to, would break the ability for the commit page to move from the reader page back into the ring-buffer. So writers must check for this condition and start dropping writes if it is detected. 

```
reader page    commit page
    		  |              |
    		  v              |
    		 +---+           |
    		 |   |<----------+
    		 |   |
    		 |   |------+
    		 +---+      |
    			    |
    			    v
    	+---+    +---+    +---+    +---+
        <---|   |--->|   |-H->|   |--->|   |--->
        --->|   |<---|   |<---|   |<---|   |<---
    	+---+    +---+    +---+    +---+
    		   ^
    		   |
    	       tail page
```

Other complex scenarios are possible. Interested readers are directed to Rostedt's design document for more information. It is quite detailed and chock full of ASCII artwork depicting ring-buffer operations. The algorithm itself is the subject of a patent application by Rostedt for Red Hat. If granted, it will be available for free software implementations under Red Hat's [patent policy](<http://www.redhat.com/legal/patent_policy.html>). 

Mathieu Desnoyers, developer of the [Linux Trace Toolkit Next Generation](<http://lttng.org/>) (LTTng), has been following the ring-buffer submission closely, as LTTng would be one of the tracing solutions expected to use the common ring-buffer. The proposed algorithm is complex, ""near that of RCU mechanisms"", he said, but unlike RCU (or the LTTng lockless buffer algorithm), no formal proof of the algorithm has been done. He agrees that lockless buffers for tracing are desirable and achievable, but he is concerned that the lack of formal verification of Rostedt's algorithm could lead to an extended period of bug chasing. That complexity has a bit of a silver lining, though, as Desnoyers [noted](<http://thread.gmane.org/gmane.linux.kernel/849409/focus=849495>) in a review of the design: ""The great news to me is that no one can say LTTng's lockless buffering algorithm is complex compared to this. ;)"" 

Two other concerns he mentioned are performance and fast user-space tracing. Rostedt's algorithm depends on being able to disable preemption, which is not possible for user-space tracing. Desnoyers said that LTTng has more compact events which he believes will allow the LTTng version to be able to handle more events per second than Rostedt's, but no real performance comparisons have, as yet, been done. Desnoyers is hopeful that he will be able to propose an alternative lockless ring-buffer implementation based on the LTTng code sometime soon, but there is the small matter of a Ph.D. dissertation to complete before that can happen. 

Rostedt is targeting the 2.6.32 kernel for merging the lockless ring-buffer, it remains to be seen if there will be objections to its inclusion. It may also have to fend off alternatives. Sooner or later, though, some kind of lockless buffering for trace events seems likely to make it into the kernel.
