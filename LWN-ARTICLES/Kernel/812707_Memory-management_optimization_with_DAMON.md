---
title: "Memory-management optimization with DAMON"
url: https://lwn.net/Articles/812707/
date: "February 20, 2020"
category: "Memory management-DAMON; Releases-5.15"
author: "By Jonathan Corbet February 20, 2020"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
February 20, 2020

To a great extent, memory management is based on making predictions: which pages of memory will a given process need in the near future? Unfortunately, it turns out that predictions are hard, especially when they are about future events. In the absence of useful information sent back from the future, memory-management subsystems are forced to rely on observations of recent behavior and an assumption that said behavior is likely to continue. The kernel's memory-management decisions are opaque to user space, though, and often result in less-than-optimal performance. A pair of patch sets from SeongJae Park tries to make memory-usage patterns visible to user space, and to let user space change memory-management decisions in response. 

At the core of this new mechanism is [the data access monitor](<https://lwn.net/ml/linux-kernel/20200217102544.29012-1-sjpark@amazon.com/>) or DAMON, which is intended to provide information on memory-access patterns to user space. Conceptually, its operation is simple; DAMON starts by dividing a process's address space into a number of equally sized regions. It then monitors accesses to each region, providing as its output a histogram of the number of accesses to each region. From that, the consumer of this information (in either user space or the kernel) can request changes to optimize the process's use of memory. 

Reality is a bit more complex than that, of course. Current hardware allows for a huge address space, most of which is unused; dividing that space into (for example) 1000 regions could easily result in all of the used address space being pushed into just a couple of regions. So DAMON starts by splitting the address space into three large chunks which are, to a first approximation, the text, heap, and stack areas. Only those areas are monitored for access patterns. 

For each region, DAMON tries to track the number of accesses. Watching every page in a region would be expensive, though, and one of the design goals of DAMON is to be efficient enough to run on production workloads. These objectives are reconciled by assuming that all pages in a given region have approximately equal access patterns, so there is no need to watch more than one of them. Thus, within each region, the "accessed" bit on a randomly selected page is cleared, then occasionally checked. If that page has been accessed, then the region is deemed to have been accessed. 

It would be nice if a process being monitored would helpfully line up its memory-access patterns to match the regions chosen by DAMON, but such cooperation is rare in real-world systems. So the layout of those equally sized regions is unlikely to correspond well with how memory is actually being used. DAMON attempts to compensate for this by adjusting the regions on the fly as the process executes. Regions showing heavy access patterns are divided into smaller areas, while those seeing little use are coalesced into larger blocks. If all this works well, the result over time should be a zeroing-in on the truly hot areas of the target process's address space. 

To control all of this, DAMON creates a set of virtual files in the debugfs filesystem. There is no access control implemented within DAMON itself, but those files are set up for root access only by default. All of the relevant parameters — target process, number of regions, and sampling and aggregation periods — can be configured by writing to those files. The resulting data can be read from debugfs; it is also possible to have the kernel write sampling data directly to a file, from which it can be processed at leisure. As an alternative, users can attach to a tracepoint to receive the data as it is generated; this makes it readily available to the `perf` tool, among other things. 

That data, however it is obtained, is essentially a histogram; each memory region is a bin and the number of hits in that bin is recorded. That data can be analyzed by hand, of course; there is also [a sample script](<https://lwn.net/ml/linux-kernel/20200217102854.30005-1-sjpark@amazon.com/>) that can feed it to [gnuplot](<http://www.gnuplot.info/>) to present the information in a more graphic form. This information, Park says, can be highly useful: 

To see the usefulness of the monitoring, we optimized 9 memory intensive workloads among them for memory pressure situations using the DAMON outputs. In detail, we identified frequently accessed memory regions in each workload based on the DAMON results and protected them with mlock() system calls. The optimized versions consistently show speedup (2.55x in best case, 1.65x in average) under memory pressure. 

That kind of speedup certainly justifies spending some time looking at a process's memory patterns. It would be even nicer, though, if the kernel could do that work itself — that is what a memory-management subsystem is supposed to be for, after all. As a step in that direction, Park has posted [a separate patch set](<https://lwn.net/ml/linux-kernel/20200218085309.18346-1-sjpark@amazon.com/>) implementing the "data access monitoring-based memory operation schemes". This mechanism allows users to tell DAMON how to respond to specific sorts of access patterns. This is done through a new debugfs file ("`schemes`") that accepts lines like: 

```
min-size max-size min-acc max-acc min-age max-age action
```

Each rule will apply to regions between `min-size` and `max-size` in length with access counts between `min-acc` and `max-acc`. These counts must have been accumulated in a region with an age between `min-age` and `max-age`. The "age" of a region is reset whenever a significant change happens; this can include the application of an action or a resizing of the region itself. 

The `action` is, at this point, a command to be passed to an [`madvise()`](<http://man7.org/linux/man-pages/man2/madvise.2.html>) call on the region; supported values are `MADV_WILLNEED`, `MADV_COLD`, `MADV_PAGEOUT`, `MADV_HUGEPAGE`, and `MADV_NOHUGEPAGE`. Actions of this type could be used to, for example, explicitly force out a region that sees little use or to request that huge pages be used for hot regions. Comments within the patch set suggest that `mlock` is also envisioned as an action, but that is not currently implemented. 

A mechanism like this has clear value when it comes to helping developers tune the memory-management subsystem for their workloads. It raises an interesting question, though: given that the kernel can be made to tune itself for better memory-management results, why isn't this capability a part of the memory-management subsystem itself? Bolting it on as a separate module might be useful for memory-management developers, who are likely interested in trying out various ideas. But one might well argue that production systems should Just Work without the need for this sort of manual tweaking, even if the tweaking is supported by a capable monitoring system. While DAMON looks like a useful tool now, users may be forgiven for hoping that it makes itself obsolete over time.
