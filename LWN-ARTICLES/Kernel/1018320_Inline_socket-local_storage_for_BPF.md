---
title: "Inline socket-local storage for BPF"
url: https://lwn.net/Articles/1018320/
date: "April 28, 2025"
category: BPF
author: "By Daroc Alden April 28, 2025 LSFMM+BPF"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Daroc Alden**  
April 28, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

Martin Lau gave a talk in the BPF track of the 2025 Linux Storage, Filesystem, Memory-Management, and BPF Summit about a performance problem plaguing the networking subsystem, and some potential ways to fix it. He works on BPF programs that need to store socket-local data; amid other improvements to the networking and BPF subsystems, retrieving that data has become a noticeable bottleneck for his use case. His proposed fix prompted a good deal of discussion about how the data should be laid out. 

One day, Lau said, Yonghong Song showed him an instruction-level profile of some kernel code from the networking subsystem. Two instructions in particular were much hotter than it seemed like they should be. In [ `bpf_sk_storage_get()`](<https://elixir.bootlin.com/linux/v6.14.3/source/net/core/bpf_sk_storage.c#L227>) (which looks up socket-local data for a BPF program), the inline function [ `bpf_local_storage_lookup()`](<https://elixir.bootlin.com/linux/v6.14.3/source/net/core/bpf_sk_storage.c#L21>) needs to dereference two pointers in order to retrieve the user data associated with a given socket. As it turns out, both of those pointer indirections were causing expensive cache misses. 

The socket-local data storage is laid out like this because the kernel can't know how much space BPF programs will need in their maps ahead of time, and so must be able to dynamically allocate the correct amount. In practice, however, the BPF programs in use at Meta, where Lau works, do not change the layout of their per-socket data frequently. One program hasn't changed the layout at all since 2021. 

[ ![\[Martin Lau\]](https://static.lwn.net/images/2025/martin-lau-lsfmmbpf-small.png) ](<https://lwn.net/Articles/1018332>)

So what if that data could be stored inline in the socket structure? Specifically, Lau proposed adding a new kernel-configuration parameter for reserving space in the structure. When set to zero, the kernel would keep the current behavior. When set to some non-zero value, allocations of socket-local data from BPF programs could be taken from the reserved space until it fills up, before falling back to the existing path. For Meta, which knows how much storage its BPF programs use per socket, this would allow the kernel to be configured with the appropriate size ahead of time, completely avoiding the double-dereference and cache misses. Reorganizing the storage like this would also allow saving 64 bytes of internal overhead from the BPF map per socket. 

One problem with the scheme is that the memory would no longer be visible to per-control-group accounting. Right now, when creating a per-socket BPF map, that memory is charged to the user-space program that created the map, Lau explained. With this proposal, it would count against the kernel instead. 

Alexei Starovoitov asked whether the array size really needed to be configured at build time; couldn't the socket structure end with a variable-length array? Lau agreed that it was possible, but thought that it would be complex to handle for all of the different types of socket. After a bit of thought, Starovoitov suggested that this might be a good fit for [ run-time constants](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e3c92e81711d>): values that are treated as constants and hard-coded into the kernel's machine code, but that can be patched at boot time. Lau said that he didn't know that was possible, but that it seemed like it could fit. 

Another developer asked what would happen if a BPF program reserved space in the inline storage, and was then reloaded by user space — would it get a new allocation (leaving the old allocation as garbage), or reuse the old allocation somehow? Lau thought that still needed to be decided, and asked for ideas. 

Andrii Nakryiko wanted to know how much space Lau intended to reserve; ""like, kilobytes?"" Lau clarified that they needed somewhere around 100 bytes, possibly less if BPF programs could be made to share data that they all need. That allayed some of Nakryiko's concerns, but he still wondered how bad a single pointer indirection would be. What if the socket structure stored a single pointer to a dynamically sized object? Data at the end of the socket structure is unlikely to be in cache anyway, he asserted. Lau disagreed, saying that it depends on how the packet has been processed so far. 

That said, one potential benefit of using non-inline storage is the ability to share data between multiple sockets, he said. For some of the BPF programs he worked with, there might only be 500 different values in a given BPF map across 500,000 sockets. If those could be combined, it could decrease cache misses and make non-inline storage less expensive. He didn't think that it would help with the current design, however, since the first dereference would still not be cached. 

Song Liu wondered if it would make more sense to compress the data — if there are only 500 variants, it should only need 9 bits. Nakryiko agreed, suggesting that the current data structures could be stored in a separate map, and the socket structures could store a small index into that map. Lau said that he had tried that, and the approach ran into some problems. 

Unfortunately, the session also ran into the end of its scheduled time before a design could be pinned down.
