---
title: Heterogeneous memory management meets EXPORT_SYMBOL_GPL()
url: https://lwn.net/Articles/757124/
date: "June 12, 2018"
category: "Copyright issues; Memory management-Heterogeneous memory management"
author: "By Jonathan Corbet June 12, 2018"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
June 12, 2018

One of the many longstanding — though unwritten — rules of kernel development is that infrastructure is not merged until at least one user for that infrastructure exists. That helps developers evaluate potential interfaces and be sure that the proposed addition is truly needed. A big exception to this rule was made when the heterogeneous memory management (HMM) code was merged, though. One of the reasons for the lack of users in this case turns out to be that many of the use cases are proprietary; that has led to some disagreements over the GPL-only status of an exported kernel symbol. 

The HMM subsystem exists to support peripherals that have direct access to system memory through their own memory-management units. It allows the ownership of ranges of memory to be passed back and forth and notifies peripherals of changes in memory mappings to keep everything working well together. HMM is not a small or simple subsystem, and bringing it into the kernel has forced a number of low-level memory-management changes. After a multi-year development process, the core HMM code was merged for the 4.14 kernel, despite the lack of any users. 

The immediate issue has to do with HMM's use of `[devm_memremap_pages()](<https://elixir.bootlin.com/linux/latest/source/kernel/memremap.c#L319>)`, which allows the mapping of pages that exist in device memory. Early versions of HMM used this function before switching to [an internal version](<https://elixir.bootlin.com/linux/latest/source/mm/hmm.c#L1026>) with some changes. Dan Williams recently posted [a patch series](<https://lwn.net/Articles/757133/>) adjusting `devm_memremap_pages()` and changing HMM to use it, getting rid of the duplicated code. That change is not controversial, but one other part of the patch set is: he changed the export declaration of `devm_memremap_pages()` to `EXPORT_SYMBOL_GPL()`. 

There are, of course, two ways to export symbols from the kernel to loadable modules, with and without the `_GPL` suffix. Symbols exported with that suffix will be unavailable to any module that does not declare a GPL-compatible license. It is a statement that, in the developers' belief, any use of those symbols will necessarily make the module a derived work of the kernel. In this case, the proposed changes will make it harder for proprietary modules to use HMM. 

Jérôme Glisse, the author of HMM, is naturally [opposed](<https://lwn.net/Articles/757135/>) to this change, since it defeats part of the purpose for HMM in the first place. Dave Airlie has also [questioned the change](<https://lwn.net/Articles/757136/>), noting that `devm_memremap_pages()` was exported normally for three years and wondering what has changed: 

If something wasn't a derived work for 3 years using that API, then it isn't a derived work now 3 years later because you changed the marker. Retrospectively changing the markers doesn't really make any sense legally or otherwise. 

Williams [responded](<https://lwn.net/Articles/757137/>) that the initial marking of the symbol was ""an oversight"" that is being corrected now. In support of the claim that any user of `devm_memremap_pages()` must be derived from the kernel, he pointed out that turning on this remapping capability changes the kernel fundamentally. The reverse of Airlie's logic also works: if a user of this functionality was a derived work of the kernel before, the non-GPL status of the export will not have changed that fact. 

Williams further [explained](<https://lwn.net/Articles/757139/>) the reasoning behind his proposed changes as: 

My concern is the long term health and maintainability of the Linux kernel. HMM exports deep Linux internals out to proprietary drivers with no way for folks in the wider kernel community to validate that the interfaces are necessary or sufficient besides "take Jerome's word for it". 

The rest of his message perhaps gets closer to the real source of this particular dispute, though: the fact that there are no in-tree users of the HM functionality. 

Every time I've pushed back on any HMM feature the response is something to the effect of, "no, out of tree drivers need this". HMM needs to grow upstream users and the functionality needs to be limited to whatever those upstream users exploit. Since there are no upstream users of HMM, we should delete it unless / until those users arrive. 

Glisse has [a response](<https://lwn.net/Articles/757141/>) to all of these complaints. HMM, he says, is meant to isolate drivers from core memory-management internals rather than tying them together. There is a user now in the form of [patches to the Nouveau driver](<https://lwn.net/Articles/752966/>) for NVIDIA GPUs; he said he hopes to get that code upstream in 4.19. And upstreaming the pieces, he said, has been ""a big chicken and egg nightmare"" with a lot of independent pieces to prepare together; that has made it hard to get the users merged along with the infrastructure. 

The merging of the Nouveau code, if and when it happens, should resolve the question of whether HMM should be in the kernel at all; it might reopen some questions about specific HMM interfaces, though. The question about the GPL-only export may prove harder to reach a conclusion on, though. There is no easy or objective standard for deciding whether the use of a specific kernel function makes a module into a derived work; it usually comes down to the judgment of the developers who wrote the code in the first place. In this case, those developers are Williams and Christoph Hellwig, who has [stated](<https://lwn.net/Articles/757183/>) that he is willing to enforce the GPL against users of `devm_memremap_pages()`. 

While a case could thus be made for changing the status of this symbol, it's not at all clear what will actually happen. Either Andrew Morton or Linus Torvalds will almost certainly end up making the final decision. It is more clear, though, that a number of developers are unhappy with the no-users status of HMM in the kernel. The most likely outcome of this particular episode may end up being a redoubling of the community's determination not to accept new subsystems into the kernel until users exist.
