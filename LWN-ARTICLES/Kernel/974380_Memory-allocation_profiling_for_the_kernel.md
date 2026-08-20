---
title: "Memory-allocation profiling for the kernel"
url: https://lwn.net/Articles/974380/
date: "May 21, 2024"
category: "Development tools-Kernel tracing"
author: "By Jonathan Corbet May 21, 2024 LSFMM+BPF"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
May 21, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

Optimizing the kernel's memory use is made much easier if developers have an accurate idea of how memory is being used, but the kernel's instrumentation is not as good as it could be. When Suren Baghdasaryan and Kent Overstreet [presented](<https://lwn.net/Articles/932402/>) their memory-allocation profiling work, which is meant to address this shortcoming, at the 2023 Linux Storage, Filesystem, Memory Management, and BPF Summit, their objective was uncontroversial but the proposed solution ran into opposition that played out at length on the mailing lists ([example](<https://lwn.net/ml/linux-kernel/20240212213922.783301-1-surenb@google.com/>)) over the last year. So it may be a bit surprising that, when the two returned to the memory-management track in the [2024 gathering](<https://events.linuxfoundation.org/lsfmmbpf/>), the controversy was gone and the discussion focused on improving details of the implementation. 

As a review: the allocation-profiling work tracks all allocations of memory in the kernel and maps them back to the code that performed the allocation. It can be used to see where memory is being used and to track down memory leaks. The profiling, in turn, relies on [code tagging](<https://lwn.net/Articles/906660/>), which inserts special structures into the code allowing locations to be identified. Both features are new to the mainline kernel. 

[![\[Suren
Baghdasaryan\]](https://static.lwn.net/images/conf/2024/lsfmm/SurenBaghdasaryan-sm.png)](<https://lwn.net/Articles/974382/>) Baghdasaryan started by saying that the patch set had been accepted into the mm-stable tree and was poised to go upstream into the mainline (that has since happened in the 6.10 merge window). The discussion on whether this code should be merged was over, so it was time to talk about what comes next. 

The main topic was reducing the memory and performance overhead of the profiling mechanism. If it is enabled, it consumes about 0.2% of the system's total memory — enough to be concerned about. It turns out that almost all of that overhead is in the [`page_ext` structures](<https://elixir.bootlin.com/linux/v6.9.1/source/include/linux/page_ext.h#L45>) used to hold the back pointer from a page of memory to the tag identifying the code where it was allocated. That pointer is used to decrement the associated counters when the page is freed. On the performance side, allocation profiling makes page allocations 40% slower, and has a smaller, 7% impact on slab allocations. 

One way of reducing that overhead would be to pack the code-tag references, of which there are 4-5,000 in the kernel. With some care, there is no need to use a 64-bit pointer for each. Instead, the references could be made smaller and, possibly, packed into the page flags, eliminating the need for the `page_ext` structure and reducing the allocation overhead. On the other hand, this approach would introduce complications with loadable modules, Baghdasaryan said. The group then spent a while discussing possible linker tricks to solve that problem without reaching any specific conclusions. 

Assuming the loadable-module problem can be solved, the allocation-profiling code would store 16-bit references rather than 64-bit ones, resulting in a 75% reduction in the memory used — for page allocations. The overhead for _slab_ allocations actually increases to 9.5%, though, suggesting that perhaps those references should not be packed. But if that 16-bit reference can be crammed into the page flags, then the memory overhead goes away completely and the performance overhead at allocation time goes from 40% to 7%. Without this additional step, he said, the packed references are not worth the extra complexity cost. 

John Hubbard was the one to ask a question that was likely on the mind of many of the developers in the room: is it really possible to find 16 free page flags to use for this purpose? Page flags have [long been in short supply](<https://lwn.net/Articles/787338/>), and developers have [had to fight hard](<https://lwn.net/Articles/335768/>) to use even a single one of them. There was not a clear answer to that question. Pasha Tatashin suggested that perhaps fewer than 16 bits would suffice for 5,000 references. There followed a winding discussion on the kernel configurations used by various distributions, their effect on the availability of page flags, and whether any of them could be changed, that did not reach any specific conclusions 

[![\[Kent
Overstreet\]](https://static.lwn.net/images/conf/2024/lsfmm/KentOverstreet-sm.png)](<https://lwn.net/Articles/974384/>) Tatashin said that it would be nice to have the ability to selectively enable and disable tags to, for example, avoid slowing down a critical network driver while profiling allocations in an unrelated subsystem. He would also like to separate accounted and unaccounted allocations; the latter, which are not charged to any specific process, represent pure overhead imposed by the kernel. Overstreet answered that the profiling could show the allocation flags used along with other information, but also asked whether it might not be better to just turn on accounting for all allocations. He acknowledged that accounting would have to be made cheaper for that to be an option. 

The allocation-profiling subsystem's path into the kernel was eased by the dropping of a number of features that it had initially included. Now the developers would like to bring some of those back, Baghdasaryan said. These include capturing more information about allocation context and dynamic fault injection (which wasn't discussed in the session; this feature allows allocation failures to be injected into specific code paths to test error handling). Some sort of selection mechanism, as requested by Tatashin earlier, is also on the list. Overstreet closed the session by saying that interest in allocation profiling (and code tags) is increasing, and that some interesting uses that he had never thought of were emerging.
