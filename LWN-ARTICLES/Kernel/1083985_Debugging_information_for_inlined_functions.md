---
title: Debugging information for inlined functions
url: https://lwn.net/Articles/1083985/
date: "July 29, 2026"
category: BPF
author: "By Daroc Alden July 29, 2026 LSFMM+BPF"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Daroc Alden**  
July 29, 2026

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2026/>)

BPF programs use [ BPF type format](<https://docs.kernel.org/bpf/btf.html>) (BTF) debugging information in order to determine how to interact with functions in the kernel. Specifically, tracing a kernel function involves finding its address in the kernel's BTF section — but that doesn't work for functions that have been inlined, and therefore don't have a single, specific address. Alan Maguire wants to add information about inlined functions to BTF in order to allow them to be traced, and led a session on that topic at the 2026 [ Linux Storage, Filesystem, Memory-Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>). 

There are more than 100,000 inlined functions in the kernel, Maguire said, spread across five times as many locations. Worse, some of them are partially inlined: called normally in some places and inlined in others. That can lead to cases today where it appears that a function was traced successfully, but some invocations were not seen. 

The good news is that the rest of the infrastructure for tracing inlined functions is already in place to enable [ kprobes](<https://docs.kernel.org/trace/kprobes.html>), which can be attached to arbitrary locations. It is just a matter of getting the data about where functions have been inlined into a usable format, Maguire stated. ""The story is actually pretty complete."" 

So, what is needed to store this information in BTF? The [ DWARF debugging format](<https://dwarfstd.org/>) already has a way to indicate inlining information, but DWARF is also difficult to work with, and doesn't have a simple way to represent the common cases. A solution for BTF should be compact, and permit deduplication, to keep the memory overhead low, Maguire said. Ideally, inlining information could be stored in a separate section of the kernel binary, or even be distributed as a separate kernel module, so that it is not loaded until it is needed. 

Concretely, Maguire proposed three new pieces of information be added to BTF. The first was inline-site-specific information about which function was inlined at each call site and how it would have been called if it had not been inlined, called the "location section". That data can't be easily deduplicated because it's specific to a given call site, so as much as possible the information should consist of pointers to data that can be deduplicated. 

The second and third pieces of information he wants to add would be the pointed-to data: a "location prototype" and "location parameter". The location prototype specifies how the inlined function's arguments are represented at the call site, as a list of pointers to location parameters, which each store how to access a single function argument. In theory, the compiler could store a given function parameter at a different location for every inlined call site (of which there are 538,090); in practice, there are a limited number of ways that the compiler will transform parameters, and many functions have compatible signatures that result in the compiler making the same choices. In the current kernel, deduplicating location prototypes results in just 57,141 distinct entries referencing only 17,535 location parameter entries in total. 

This means that the location sections take up most of the added data. Overall, the additions to BTF that Maguire proposes would come to approximately 11MB of additional data, or about 21 bytes per inlined call site. When pulled out into a separate kernel module and compressed, the total amount of data goes down to 3.5MB. 

Maguire then went through an example of how information about a specific function would be stored. Consider this function: 

```
int foo(int a, void *b, bool c);
```

If the compiler chose to inline `foo()` in a way that eliminates `a` as unused, promotes `b` to be passed in a register, and determines that `c` is a constant, the BTF representation would be a single location section entry storing the BTF type ID of `foo()`, the offset of the call site from the kernel's base address in memory, and a pointer to the location-prototype entry. That entry becomes a length-tagged array of references to location-parameter entries. The first entry is null, indicating that `a` cannot be recovered. The second points to a location parameter entry with a flag that indicates the value is contained in a register, and then the specific register number of `b`. The last location-parameter entry has a different flag specifying that it is a constant, and then the value of the constant. 

Alexei Starovoitov asked what order the location-section entries would be stored in, since the kernel will have to search through them to find the right entry when setting up tracing. At first, Maguire thought that sorting by the address of the call site would be best, but after looking at how the data would be used, he decided to sort the entries by function name, instead. That way, tracers that are looking for information on a particular function can find it quickly with a binary search. 

There were two prerequisite problems with adding inlining information to BTF that Maguire covered. For one thing, the number of location-section entries required increasing the size of the length field of the containing structure to 24 bits. More seriously, some of the existing tooling for processing BTF did not cope well with the presence of new tags that it doesn't know how to read. That is a problem that comes up any time BTF is extended with new kinds of information, and part of how DWARF ended up becoming so brittle. To address that, Maguire added information on how to parse BTF into BTF itself. Now, any tool that can read that meta-information can correctly skip past any new tags that it does not understand. 

Maguire also had to update the [ poke-a-hole (pahole) utility](<https://lwn.net/Articles/335942/>) to handle properly sorting the new kinds of BTF tags. That ended up being a bit complicated, but it should be working for normal kernel builds. Andrii Nakryiko asked about how it would handle out-of-tree modules. Those modules would need [ resilient module BTF](<https://lwn.net/Articles/978317/>) that could be explicitly relocated when loaded into a running kernel, Maguire explained. His design allows that, but it complicates the build somewhat. There was a bit of back-and-forth about the changes that would be needed to support out-of-tree modules elegantly, but ultimately no changes were agreed upon. 

At the time of writing, Maguire's [ patch set](<https://lwn.net/ml/bpf/20251008173512.731801-1-alan.maguire@oracle.com/>) has not been merged. It is clearly useful to be able to trace inlined and partially inlined functions; the question is whether it is worth the complexity and memory overhead. Time will tell.
