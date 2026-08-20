---
title: "Toward a real \"too small to fail\" rule"
url: https://lwn.net/Articles/964793/
date: "March 18, 2024"
category: "Memory management-Page allocator"
author: "By Jonathan Corbet March 18, 2024"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
March 18, 2024

Kernel developers have long been told that any attempt to allocate memory might fail, so their code must be prepared for memory to be unavailable. Informally, though, the kernel's memory-management subsystem implements a policy whereby requests below a certain size will not fail (in process context, at least), regardless of how tight memory may be. A recent discussion on the linux-mm list has looked at the idea of making [the "too small to fail" rule](<https://lwn.net/Articles/723317/>) a policy that developers can rely on. 

The kernel is unable to use virtual memory, so it is strictly bound by the amount of physical memory in the system. Depending on what sort of workload is running, that memory could be tied up in various ways and unavailable for allocation elsewhere. Allowing allocation requests to fail gives the kernel the freedom to avoid making things worse when memory pressure is high. 

There are some downsides to failing an allocation request, of course. Whatever operation needed that memory will also be likely to fail, and that failure will probably propagate out to user space, resulting in disgruntled users. There is also a significant chance that the kernel will not handle the allocation failure properly, even if the developers have been properly diligent. Failure paths can be hard to test; many of those paths in the kernel may never have been executed and, as a consequence, many are likely to have bugs. Unwinding an operation halfway through can be a complex business, which is not the kind of task one wants to see entrusted to untested code. 

Recently, Neil Brown [started a sub-thread](<https://lwn.net/ml/linux-mm/170925937840.24797.2167230750547152404@noble.neil.brown.name/>) in a wide-ranging discussion on memory-management policies by suggesting a reconsideration of the rules around `GFP_KERNEL` allocations. Currently, programmers have to be prepared for those calls to fail, even if, in fact, the kernel will not fail small allocations. Brown proposed to make the "too small to fail" behavior a documented rule, at least for allocations below a predefined size. `GFP_KERNEL` allocations are allowed to sleep, he said, and thus have access to all of the kernel's machinery for freeing memory. In the worst case, the out-of-memory (OOM) killer can be summoned to remove a few processes from the system. If this code is unable to create some free memory, he said, ""the machine is a goner anyway"". If, instead, `GFP_KERNEL` allocations would always succeed, he concluded, it ""would allow us to remove a lot of untested error handling code"". 

Kent Overstreet [objected](<https://lwn.net/ml/linux-mm/wpof7womk7rzsqeox63pquq7jfx4qdyb3t45tqogcvxvfvdeza@ospqr2yemjah/>) to this idea, though. It is common, he said, for kernel code to attempt to allocate memory to carry out a task efficiently, but to be able to fall back to a slower approach if the memory is unavailable; such mechanisms will not work if memory requests do not fail. Even worse, the kernel's efforts to satisfy such requests may worsen performance elsewhere in the system. Without allocation failure, there is no signal to indicate that memory is tight; the implementation of memory overcommit for user space has, he said, made it impossible to use memory efficiently there. 

The real solution, he said, is proper testing of all those error paths; ""relying on the OOM killer and saying that because [of] that now we don't have to write and test your error paths is a lazy cop out"". James Bottomley [disagreed](<https://lwn.net/ml/linux-mm/a43cf329bcfad3c52540fe33e35e2e65b0635bfd.camel@HansenPartnership.com/>), pointing out that the OOM killer only runs in extreme situation, and that error paths are a problem. ""Error legs are the least exercised and most bug, and therefore exploit-prone pieces of code in C. If we can get rid of them, we should."" Overstreet [was unimpressed](<https://lwn.net/ml/linux-mm/3bykct7dzcduugy6kvp7n32sao4yavgbj2oui2rpidinst2zmn@e5qti5lkq25t/>): ""Having working error paths is _basic_, and learning how to test your code is also basic. If you can't be bothered to do that you shouldn't be writing kernel code."" 

Dave Chinner, instead, was [enthusiastically supportive](<https://lwn.net/ml/linux-mm/ZeFtrzN34cLhjjHK@dread.disaster.area/>) of the idea. The XFS filesystem, he said, was originally developed for a kernel (IRIX) that provided a guarantee for allocations. ""A simple change to make long standing behaviour an actual policy we can rely on means we can remove both code and test matrix overhead - it's a win-win IMO."" 

Brown later [modified his proposal](<https://lwn.net/ml/linux-mm/170933687972.24797.18406852925615624495@noble.neil.brown.name/>) slightly, noting that changing the semantics of `GFP_KERNEL` might cause problems for existing code. Instead, perhaps, `GFP_KERNEL` could be deprecated entirely in favor of a new set of allocation types. He later [suggested](<https://lwn.net/ml/linux-mm/170950594802.24797.17587526251920021411@noble.neil.brown.name/>) this hierarchy: 

  * `GFP_NOFAIL` would explicitly request the "cannot fail" behavior and could, as a result, wait a long time for an allocation request to be fulfilled. 
  * `GFP_KILLABLE` would be the same as `GFP_NOFAIL`, with the exception that requests will fail in the presence of a fatal signal. 
  * `GFP_RETRY` would make multiple attempts to satisfy an allocation request, but would eventually fail if no progress is made. 
  * `GFP_NO_RETRY` would only allow a single attempt (which could still sleep) at allocating memory, after which the request would fail. 
  * `GFP_ATOMIC` would not sleep at all (which is the current behavior). 

Given these options, he said, `GFP_KERNEL` could go: 

> I don't see how "GFP_KERNEL" fits into that spectrum. The definition of "this will try really hard, but might fail and we can't really tell you what circumstances it might fail in" isn't fun to work with. 

Overstreet [responded](<https://lwn.net/ml/linux-mm/aownm3xt34rju5tvhsrkbcurls2vlyzueamreiqd3uuompyioj@x3wkk7w6iroy/>), once again, that these changes were not needed: ""We just need to make sure error paths are getting tested - we need more practical fault injection, that's all."" Chinner, instead, [commented](<https://lwn.net/ml/linux-mm/ZeUTyxYFS6kGoM1h@dread.disaster.area/>) that `GFP_KILLABLE` and `GFP_RETRY` were essentially the same thing; Brown [responded](<https://lwn.net/ml/linux-mm/170951501074.24797.10807279234722357224@noble.neil.brown.name/>) that, perhaps, the key distinguishing feature of those allocation types is that they would not invoke the OOM killer; perhaps both of them could be replaced with a single `GFP_NOOOM` type. ""We might need a better name than GFP_NOOOM :-)"". 

Matthew Wilcox [raised](<https://lwn.net/ml/linux-mm/ZeUXORziOwkuB-tP@casper.infradead.org/>) a different sort of objection. The proper allocation policy for any given request depends on the context in which the request is made; a function called from an interrupt handler has fewer options available than one running in process context. Sometimes, the code that knows about that context is several steps back in the call chain from the function doing the allocation. The way to set the allocation type, he said, is through the use of context flags applied to the current thread. 

Brown, though, [pointed out](<https://lwn.net/ml/linux-mm/170951563963.24797.10928820769529800242@noble.neil.brown.name/>) that this context is not the full picture. If code has been written assuming `GFP_NOFAIL` behavior, it would be incorrect to allow the context to change an allocation into one that could fail: ""context cannot add error handling"". 

Vlastimil Babka [worried](<https://lwn.net/ml/linux-mm/a7862cf1-1ed2-4c2c-8a27-f9d950ff4da5@suse.cz/>) that deprecating `GFP_KERNEL` would be an unending task. Instead, guaranteeing "too small to fail" could be done quickly, and modifying specific call sites to allow allocation failure would be a relatively easy task, so he suggested taking that path. Brown, though, [answered](<https://lwn.net/ml/linux-mm/171028138478.13576.3004333623297072625@noble.neil.brown.name/>) that [removing the big kernel lock](<https://lwn.net/Articles/424657/>) also took a long time: ""I don't think this is something we should be afraid of"". Since redefining `GFP_KERNEL` also implies removing error-handling code, he said, it should still be handled one call site at a time. 

The discussion wound down at about this point, but there is a good chance that we'll be hearing these ideas again. The kernel, for all practical purposes, already implements `GFP_NOFAIL` behavior for allocations of eight pages or less. Turning the behavior into a guarantee would allow for significant simplification and the removal of a lot of untested code. That is an idea with significant appeal; it would be surprising if it did not come up at the [Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>) in May.
