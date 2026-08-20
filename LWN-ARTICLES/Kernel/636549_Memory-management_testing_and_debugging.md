---
title: "Memory-management testing and debugging"
url: https://lwn.net/Articles/636549/
date: "March 16, 2015"
category: "Development tools-Testing; KASan"
author: "By Jonathan Corbet March 16, 2015 LSFMM 2015"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
March 16, 2015

* * *

[LSFMM 2015](<https://lwn.net/Articles/lsfmm2015/>)

Memory-management problems can be hard to identify and track down; this is true for bugs that affect either correctness or performance. Quite a bit of work has been done in recent years to develop tools that can help with this task, though. The 2015 LSFMM gathering had a number of sessions dedicated to this area; like a large array on a virtual-memory system, though, they were scattered throughout the program. This article provides a virtual view of the entire discussion in one place. 

#### Testing

[![\[Davidlohr Bueso\]](https://static.lwn.net/images/conf/2015/lsfmm/DavidlohrBueso2-sm.jpg)](<https://lwn.net/Articles/636697/>) Davidlohr Bueso started a session on testing by saying that he has been working on improving the [mmtests](<https://github.com/gormanm/mmtests>) benchmark suite to improve its ability to detect changes across kernel versions. To that end, he has looked at a couple of test suites that are being used in academia: [Mosbench](<http://pdos.csail.mit.edu/mosbench/>) and [Parsec](<http://parsec.cs.princeton.edu/>). There were questions about how well these tests worked for testing the kernel in particular, but, Davidlohr said, these suites do contain some useful tests. 

Andi Kleen said there is a new suite out there that is promising despite being named, inevitably, "cloudbench." 

Davidlohr asked if anybody else had workload tests that they would like to contribute to mmtests. Laura Abbott said that she would like to see a good set of tests for mobile systems. Scalability tests, she said, tend to be oriented toward scaling up, but mobile developers need tests that focus on scaling down. 

Hard conclusions from this session were hard to come by; Davidlohr will continue to work on integrating and documenting other tests aimed at memory-management scalability. 

#### Debugging

Memory-management debugging was the topic of another session run by Dave Jones, Sasha Levin, and Dave Hansen. Dave Hansen started off by saying that, while developers have added a number of debugging features to the [![\[Dave Hansen\]](https://static.lwn.net/images/conf/2015/lsfmm/DaveHansen-sm.jpg)](<https://lwn.net/Articles/636699/>) memory-management subsystem, they have so far left an important technology on the table. He was talking about [Intel's MPX mechanism](<https://lwn.net/Articles/582712/>), which is able to check pointer accesses and ensure, in hardware, that they don't go outside a set of defined boundaries. The nice thing about MPX is that it has almost no runtime cost, so it can be enabled on production systems. 

Of course, developers may have some excuse for not making much use of MPX so far. It requires the (not yet released) GCC 5 compiler to instrument code properly, and hardware that actually implements MPX is not yet available. So, he said, there is still time to get our act together. 

There was some immediate interest in using MPX with the slab allocator in the kernel. That would take some work, though, since the kernel would have to be changed to load the appropriate MPX registers before accessing a given slab object. Christoph Lameter asked if access to _all_ slab objects could be monitored with MPX. It turns out that there's a small practical difficulty there: a typical running kernel has many thousands of slab-allocated objects, but there are only four sets of registers in the MPX hardware. So tracking more than four objects requires juggling information into and out of those registers. 

Peter Zijlstra suggested that MPX could be applied to the kernel stack. It is not clear, though, that MPX-based stack checking would provide advantages over the explicit stack-overflow checks done in the kernel now. Still, it may be possible to dedicate one of the registers to the kernel stack and gain some extra protection. 

Andy Lutomirski asked if the MPX registers could be written to while running in atomic context. That turns out to be tricky, since setting up these registers involves doing a memory allocation. Andy also suggested that [![\[Sasha Levin\]](https://static.lwn.net/images/conf/2015/lsfmm/SashaLevin-sm.jpg)](<https://lwn.net/Articles/636700/>) MPX could be used to block direct access to user-space addresses from the kernel. Laura asked about checking of DMA operations, but MPX only applies to accesses from the CPU. 

Sasha shifted the discussion to the `VM_BUG_ON()` macro. This macro, which comes in a few variants, dumps out a bunch of information specific to the memory-management subsystem; it is thus useful for identifying memory-management bugs. Sasha would like to add more `VM_BUG_ON()` instances in the kernel, but he is worried about complaints of false positives. These complaints have kept debugging code out in the past; the result, he said, was that users suffered from a number of race conditions that could otherwise have been caught. 

There was some talk about additional information that could be printed out by `VM_BUG_ON()`, but few conclusions. It was suggested that a full kernel memory dump would be helpful — but that, of course, is rather a large amount of data to print into the kernel log. Dave Jones would, instead, like more information about how the system got into the bad state; that would require adding some sort of transaction log. It was suggested that Intel's upcoming [Processor Trace functionality](<https://software.intel.com/en-us/blogs/2013/09/18/processor-tracing>) could be helpful in this regard. 

Dave Hansen then asked if there were any developers with sets of memory-management tracepoints that could be considered for merging? It seems that some exist, but Andi said that, rather than adding more tracepoints now, it would be better to focus on improving the documentation of existing tracepoints. Andrea Arcangeli questioned the value of [![\[Dave Jones\]](https://static.lwn.net/images/conf/2015/lsfmm/DaveJones-sm.jpg)](<https://lwn.net/Articles/636701/>) memory-management tracepoints in general; he does his memory-management development on virtualized systems and wonders why anybody would do anything else. When a system is run under virtualization, it can be examined with an ordinary debugger. But others argued that there are a lot of problems that only show up on bare-metal systems, so there will always be a place for debugging infrastructure that works in that environment. 

Fernando Vasquez Cao noted that his group uses SystemTap heavily for memory-management debugging. Among other things, it is handy for injecting faults at specific locations, making it easier to get at hard-to-reproduce problems. Dave Jones agreed that the tools have made life better; it is, he said, a miracle that we were able to solve anything five years ago. He also wondered why there was not more use of the existing [fault-injection framework](<https://lwn.net/Articles/209257/>); when he turns it on, he said, "everything breaks," so he concludes that nobody else is doing so. Fernando responded that the injection framework does not allow sufficiently specific fault injection. Besides, he said, when you turn it on everything else breaks, making it hard to focus on the specific problem at hand. It was agreed that somebody (currently unnamed) should fix those problems. 

#### KASan

One tool that has been merged relatively recently is the [kernel address sanitizer](<https://lwn.net/Articles/612153/>) (or KASan). This tool uses a "shadow memory" array to track which memory the kernel should legitimately be accessing; it can then throw an error whenever the kernel goes out of bounds. KASan developer Andrey Ryabinin led a session on this tool and how it might be improved. 

The first idea that came out was to enable KASan to properly validate accesses to memory obtained with `vmalloc()`. Doing so would require putting hooks into `vmalloc()` itself and creating a new, [![\[Andrey Ryabinin\]](https://static.lwn.net/images/conf/2015/lsfmm/AndreyRyabinin-sm.jpg)](<https://lwn.net/Articles/636702/>) dynamic shadow memory array. The amount of work required is not huge; it is much like tracking slab allocations, except that shadow memory for slab can be allocated at boot time. There were, unsurprisingly, no objections, so this work should go forward soon. 

A slightly trickier problem is memory that is freed and quickly reallocated to a new user. That memory looks fine to KASan, but quick reallocation can mask use-after-free bugs in the code that previously owned it. The proposed solution here is to put freed memory into a "quarantine" area for a period, delaying its availability to the rest of the system. Memory would emerge from quarantine after a defined period; alternatively, a shrinker could be used to remove memory from quarantine when the system starts to run low. There are concerns that delaying free operations in this way could create a certain amount of memory fragmentation. Andrey is not quite sure how to move forward with this feature, and the group did not appear to have a lot of fresh ideas to share. 

Then there is the possibility of catching reads of uninitialized memory. It is possible to get the compiler to instrument code to make this testing possible, but the results include a lot of false positives that are hard to get rid of. Among other things, memory initialized in assembly code must be annotated manually. Andrey has tried doing this and found the result difficult to support. He's afraid that developers will turn the feature on, see all the false positives, and just give up on the whole thing. 

Another possibility is using KASan to find data races; there are some tools out there to help with this now. But, he said, it involves some "crazy overhead" — four bytes of shadow memory for every byte of normal memory. There's also a need for a lot of manual annotation; large numbers of false positives are also a problem. The end result is that this feature does not appear to be useful for now. 

Other ideas for the more distant future include a quarantine for the page allocator (and not just the slab allocator), and the instrumentation of some inline assembly operations like the atomic bit operators. 

Sasha made a plea for developers to enable KASan when they are running their own tests. It has turned up a lot of bugs, he said; the code is in the upstream kernel, it's easy to turn on, and the overhead is low. The only catch is that GCC 5 is needed to gain all of the features, though 4.9 works with reduced functionality. 

The final question in this session was: now that we have KASan, is there still a need to maintain the older [kmemcheck](<https://lwn.net/Articles/260068/>) utility? Kmemcheck only works on single-processor systems, it is painful to use, and it is slow. It seems that nobody is actually making use of it. The consensus of the group was that kmemcheck should be removed. (It should be noted that Sasha's [attempt to implement this decision](<https://lwn.net/Articles/636216/>) ran into some opposition from developers who still use kmemcheck, so it may stay around for a while yet).
