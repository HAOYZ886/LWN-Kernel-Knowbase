---
title: Fun with tracepoints
url: https://lwn.net/Articles/346470/
date: "August 12, 2009"
category: "Development tools-Kernel tracing; Performance monitoring; Tracing"
author: "By Jonathan Corbet August 12, 2009"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
August 12, 2009

Tracepoints are a marker within the kernel source which, when enabled, can be used to hook into a running kernel at the point where the marker is located. They can be used by a number of tools for kernel debugging and performance problem diagnosis. One of the advantages of the DTrace system found in Solaris is the extensive set of well-documented tracepoints in the kernel (and beyond); they allow administrators and developers to monitor many aspects of system behavior without needing to know much about the kernel itself. Linux, instead, is rather late to the tracepoint party; mainline kernels currently feature only a handful of static tracepoints. Whether that number will grow significantly is still a matter of debate within the development community. 

LWN last [looked at the tracepoint discussion](<http://lwn.net/Articles/330402/>) in April. Since then, the disagreement has returned with little change. The catalyst this time was Mel Gorman's [page allocator tracepoints patch](<http://lwn.net/Articles/345727/>), which further instruments the memory management layer. The mainline kernel already contains tracepoints for calls to functions like `kmalloc()`, `kmem_cache_alloc()`, and `kfree()`. Mel's patch adds tracepoints to the low-level page allocator, in places like `free_pages_bulk()`, `__rmqueue_fallback()`, and `__free_pages()`. These tracepoints give a view into how the page allocator is performing; they'll inform a suitably clueful user if fragmentation is growing or pages are being moved between processors. Also included is a postprocessing script which uses the tracepoint data to create a list of which processes on the system are putting the most stress on the memory management code. 

As has happened before, Andrew Morton [questioned the value](<https://lwn.net/Articles/346483/>) of these tracepoints. He tends not to see the need for this sort of instrumentation, seeing it instead as debugging code which is generally useful to a single developer. Beyond that, Andrew asks, why can't the relevant information be added to `/proc/vmstat`, which is an established interface for the provision of memory management information to user space? 

There are a couple of answers to that question. One is that `/proc/vmstat` has a number of limitations; it cannot be used, for example, to monitor the memory-management footprint of a specific set of processes. It is, in essence, pre-cooked information about memory management in the system as a whole; if a developer needs information which cannot be found there, that information will be almost impossible to get. Tracepoints, instead, provide much more specific information which can be filtered to give more precise views of the system. Mel bashed out one [demonstration](<https://lwn.net/Articles/346484/>): a SystemTap script which uses the tracepoints to create a list of which processes are causing the most page allocations. 

Ingo Molnar posted [a lengthy set of examples](<https://lwn.net/Articles/346487/>) of what could be done with tracepoints; some of these were later taken by Mel and incorporated into [a document on simple tracepoint use](<https://lwn.net/Articles/346488/>). These examples merit a look; they show just how quickly and how far the instrumentation of the Linux kernel (and associated tools) have developed. 

One of the key secrets for quick use of tracepoints is the `perf` tool which is shipped with the kernel as of 2.6.31-rc1. This tool was written as part of the performance monitoring subsystem; it can be used, for example, to run a program and report on the number of cache misses sustained during its execution. One of the features slipped into the performance counter subsystem was the ability to treat tracepoint events like performance counter events. One must set the `CONFIG_EVENT_PROFILE` configuration option; after that, `perf` can work with tracepoint events in exactly the same way it manages counter events. 

With that in place, and a working `perf` binary, one can start by seeing which tracepoint events are available on the system: 

```
$ perf list
          ...
          ext4:ext4_sync_fs                        [Tracepoint event]
          kmem:kmalloc                             [Tracepoint event]
          kmem:kmem_cache_alloc                    [Tracepoint event]
          kmem:kmalloc_node                        [Tracepoint event]
          kmem:kmem_cache_alloc_node               [Tracepoint event]
          kmem:kfree                               [Tracepoint event]
          kmem:kmem_cache_free                     [Tracepoint event]
          ftrace:kmem_free                         [Tracepoint event]
          ...
```

How many `kmalloc()` calls are happening on a system? The question can be answered with: 

```
$ perf stat -a -e kmem:kmalloc sleep 10
    
         Performance counter stats for 'sleep 10':
    
               4119  kmem:kmalloc            
    
         10.001645968  seconds time elapsed
```

So your editor's mostly idle system was calling `kmalloc()` almost 420 times per second. The `-a` option gives whole-system results, but perf can also look at specific processes. Monitoring allocations during the building of the `perf` tool gives: 

```
$ perf stat -e kmem:kmalloc make
          ...
     Performance counter stats for 'make':
    
               5554  kmem:kmalloc            
    
      2.999255416  seconds time elapsed
```

More detail can be had be recording data and analyzing it afterward: 

```
$ perf record -c 1 -e kmem:kmalloc make
          ...
        $ perf report
        # Samples: 6689
        #
        # Overhead          Command                         Shared Object  Symbol
        # ........  ...............  ....................................  ......
        #
          19.43%             make  /lib64/libc-2.10.1.so                 [.] __getdents64
          12.32%               sh  /lib64/libc-2.10.1.so                 [.] __execve
          10.29%              gcc  /lib64/libc-2.10.1.so                 [.] __execve
           7.53%              cc1  /lib64/libc-2.10.1.so                 [.] __GI___libc_open
           5.02%              cc1  /lib64/libc-2.10.1.so                 [.] __execve
           4.41%               sh  /lib64/libc-2.10.1.so                 [.] __GI___libc_open
           3.45%               sh  /lib64/libc-2.10.1.so                 [.] fork
           3.27%               sh  /lib64/ld-2.10.1.so                   [.] __mmap
           3.11%               as  /lib64/libc-2.10.1.so                 [.] __execve
           2.92%             make  /lib64/libc-2.10.1.so                 [.] __GI___vfork
           2.65%              gcc  /lib64/libc-2.10.1.so                 [.] __GI___vfork
```

Conclusion: the largest source of `kmalloc()` calls in a simple compilation process is `getdents()`, called from `make`, followed by the `execve()` calls needed to run the compiler. 

The `perf` tool can take things further; it can, for example, generate call graphs and disassemble the code around specific performance-relevant points. See Ingo's mail and Mel's document for more information. Even then, we're just talking about statistics on tracepoints; there is a lot more information available which can be used in postprocessing scripts or tools like SystemTap. Suffice to say that tracepoints open a lot of possibilities. 

The obvious question is: was Andrew impressed by all this? Here's [his answer](<https://lwn.net/Articles/346521/>): 

So? The fact that certain things can be done doesn't mean that there's a demand for them, nor that anyone will _use_ this stuff. 

As usual, we're adding tracepoints because we feel we must add tracepoints, not because anyone has a need for the data which they gather. 

He suggested that he would be happier if the new tracepoints could be used to phase out `/proc/vmstat` and `/proc/meminfo`; that way there would not be a steadily-increasing variety of memory management instrumentation methods. Removing those files is problematic for a couple of reasons, though. One is that they form part of the kernel ABI, which is not easily broken. It would be a multi-year process to move applications over to a different interface and be sure there were no more users of the `/proc` files. Beyond that, though, tracepoints are good for reporting events, but they are a bit less well-suited to reporting the current state of affairs. One can use a tracepoint to see page allocation events, but an interface like `/proc/vmstat` can be more straightforward if one simply wishes to know how many pages are free. There is space, in other words, for both styles of instrumentation. 

As of this writing, nobody has made a final pronouncement on whether the new tracepoints will be merged. Andrew has made it clear, though, that, despite his concerns, he's not firmly opposing them. There is enough pressure to get better instrumentation into the kernel, and enough useful things to do with that instrumentation, that, one assumes, more of it will go into the mainline over time.
