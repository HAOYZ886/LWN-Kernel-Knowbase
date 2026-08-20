---
title: "Kswapd and high-order allocations"
url: https://lwn.net/Articles/101230/
date: "September 8, 2004"
category: "Kswapd; Memory management-Large allocations; Networking-Nonlinear packets v. large allocations; procbuddyinfo"
author: 
---

The core memory allocation mechanism inside the kernel is page-based; it will attempt to find a certain number of contiguous pages in response to a request (where "a certain number" is always a power of two). After the system has been running for a while, however, "higher-order" allocations requiring multiple contiguous pages become hard to satisfy. The virtual memory subsystem fragments physical memory to the point that the free pages tend to be separated from each other. 

Curious readers can query `/proc/buddyinfo` to see how fragmented the currently free pages are. On a 1GB system, your editor currently sees the following: 

```
Node 0, zone   Normal 258 9 5 0 1 2 0 1 1 0 0
```

On this system, 258 single pages could be allocated immediately, but only nine contiguous pairs exist, and only five groups of four pages can be found. If something comes along which needs a lot of higher-order allocations, the available memory will be exhausted quickly, and those allocations may start to fail. 

Nick Piggin has recently [looked at this issue](<https://lwn.net/Articles/100877/>) and found one area where improvements can be made. The problem is with the `kswapd` process, which is charged with running in the background and making free pages available to the memory allocator (by evicting user pages). The current `kswapd` code only looks at the number of free pages available; if that number is high enough, `kswapd` takes a rest regardless of whether any of those pages are contiguous with others or not. That can lead to a situation where high-order allocations fail, but the system is not making any particular effort to free more contiguous pages. 

Nick's patch is fairly straightforward; it simply keeps `kswapd` from resting until a sufficient number of higher-order allocations are possible. 

It has been pointed out, however, that the approach used by `kswapd` has not really changed: it chooses pages to free without regard to whether those pages can be coalesced into larger groups or not. As a result, it may have to free a great many pages before it, by chance, creates some higher-order groupings of pages. In prior kernels, no better approach was possible, but 2.6 includes the reverse-mapping code. With reverse mapping, it should be possible to target contiguous pages for freeing and vastly improve the system's performance in that area. 

Linus's [objection](<https://lwn.net/Articles/101238/>) to this idea is that it overrides the current page replacement policy, which does its best to evict pages which, with luck, will not be needed in the near future. Changing the policy to target contiguous blocks would make higher-order allocations easier, but it could also penalize system performance as a whole by throwing out useful pages. So, says Linus, if a "defragmentation" mode is to be implemented at all, it should be run rarely and as a separate process. 

The other approach to this problem is to simply avoid higher-order allocations in the first place. The switch to 4K kernel stacks was a step in this direction; it eliminated a two-page allocation for every process created. In current kernels, one of the biggest users of high-order allocations would appear to be high-performance network adapter drivers. These adapters can handle large packets which do not fit in a single page, so the kernel must perform multi-page allocations to hold those packets. 

Actually, those allocations are only required when the driver (and its hardware) cannot handle "nonlinear" packets which are spread out in memory. Most modern hardware can do scatter/gather DMA operations, and thus does not care whether the packet is stored in a single, contiguous area of memory. Using the hardware's scatter/gather capabilities requires additional work when writing the driver, however, and, for a number of drivers, that work has not yet been done. Addressing the high-order allocation problem from the demand side may prove to be far more effective than adding another objective to the page reclaim code, however.
