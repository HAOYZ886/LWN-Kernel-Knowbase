---
title: "A new DMA-mapping API"
url: https://lwn.net/Articles/1020437/
date: "May 15, 2025"
category: Direct memory access
author: "By Jake Edge May 15, 2025 LSFMM+BPF"
---

By **Jake Edge**  
May 15, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

Leon Romanovsky began his session at the 2025 Linux Storage, Filesystem, Memory Management, and BPF Summit (LSFMM+BPF) by explaining that the improved DMA-mapping API that he has been working on is a group effort. He, Chaitanya Kulkarni, Christoph Hellwig, Jason Gunthorpe, and others are proposing to modernize the API and to ""make it more suitable for current kernels"". He told the assembled storage and filesystem developers that the progress on the proposal has stalled, but that it was the basis for further work in various areas, so he hoped to find a way to move forward with it. 

The existing DMA API is based on [`struct page`](<https://elixir.bootlin.com/linux/v6.14.6/source/include/linux/mm_types.h#L35>), which is fine, but using DMA requires scatter-gather (SG) lists, so there is a lot of conversion between the two formats. In addition, many DMA users have their own formats for the data being transferred, leading to more conversions (between SG and native formats). 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

A [`struct scatterlist`](<https://elixir.bootlin.com/linux/v6.14.6/source/include/linux/scatterlist.h#L11>) has two fields that are supposed to contain a CPU address (`page_link`) and a DMA address (`dma_address`) but various (ab)uses of the `scatterlist` have changed the meanings of those fields. For [peer-to-peer DMA](<https://lwn.net/Articles/767281/>), `page_link` is a synthetic CPU address used to get information from the page structure, he said. For [dma-buf](<https://docs.kernel.org/driver-api/dma-buf.html>) usage, `page_link` is null and `dma_address` is synthetic in order to access the device-private memory. Hellwig pointed out that the dma-buf usage is explicitly invalid according to the documentation, but ""the dma-buf people did it anyway and refused to fix it"". 

[ ![\[Leon Romanovsky\]](https://static.lwn.net/images/2025/lsfmb-romanovsky-sm.png) ](<https://lwn.net/Articles/1021009/>)

So the [proposal](<https://lwn.net/ml/all/cover.1738765879.git.leonro%40nvidia.com/>) (which was v7 during the summit, but is now on [v11](<https://lwn.net/ml/all/cover.1746424934.git.leon%40kernel.org/>)) is ""very simple"", Romanovsky said. There is a new set of APIs that is based on a careful refactoring of the DMA-mapping API; it is meant to be bug-for-bug compatible with the existing API. Instead of having to use SG lists, DMA users will be able to directly manage their own I/O virtual address (IOVA) space, which is provided by the I/O memory-management unit (IOMMU). The new API will optimize for DMA using the IOMMU path; systems without an IOMMU can fall back to using the existing API. 

He briefly described the API, which is described more fully in an [LWN article](<https://lwn.net/Articles/997563/>) from November. It starts with a call to `dma_iova_try_alloc()` to allocate the IOVA range, then calls to `dma_iova_link()` to map memory into the range. Once all of the memory of interest has been mapped, `dma_iova_sync()` can be called; rather than synchronize on each mapping, that is an optimization to only do the expensive synchronization operation once for the whole range. 

He and Hellwig converted three or four subsystems (""depending on how you count"") to use the new API as part of the patch set, Romanovsky said. The easiest of those to follow is the [virtual function I/O](<https://www.kernel.org/doc/html/latest/driver-api/vfio.html>) (VFIO) live-migration code. He showed the before and after code; currently there are three separate loops over lengthy SG lists representing large areas of memory, which is ""a huge amount of work just to make sure we are getting back DMA addresses"" that can be used to program the hardware. Using the proposed API, that will be done in a single loop, if the optimized path can be used; if not, it will fall back to the existing mechanism. 

This work is part of a large roadmap. The idea is to eventually remove the use of SG lists for dma-buf and then for other DMA users. The roadmap is just a set of ideas at this point, no concrete work has been done while awaiting the new API. 

But he showed a [response](<https://lwn.net/ml/all/d64a40d9-0c5b-45b6-a95c-d428a4dd9640@arm.com/>) from DMA maintainer Robin Murphy that left Romanovsky ""unclear how to proceed now"". Earlier in the thread, Murphy [explicitly rejected](<https://lwn.net/ml/all/1166a5f5-23cc-4cce-ba40-5e10ad2606de@arm.com/>) the patches, but in Romanovsky's opinion, there are ""no technical objections"" that have not been addressed. He highlighted the sentence ending Murphy's final email in the thread, which does not seem to lead in a direction toward resolution: 

> And there is no obligation for maintainers to accept code with obvious significant issues just because they don't have the time or inclination to personally engage in trying to fix said issues. 

Dan Williams suggested that a way to help break the logjam would be to point out which features that developers want are being blocked. For example, some [confidential computing features are stuck](<https://lwn.net/ml/all/6801a8e3968da_71fe29411@dwillia2-xfh.jf.intel.com.notmuch/>) waiting for the new DMA API. Simply listing additional plans to switch away from SG lists does not demonstrate the use cases that need those changes; Williams suggested making those use cases visible in order to make it ""extremely clear"" that the new API is needed. 

David Howells said that he was looking forward to removing SG-list support from the crypto layer. There is a lot of SG-list creation for no real gain when encrypting and decrypting network traffic using DMA; he doesn't think the crypto layer actually needs SG lists for what it does. James Bottomley said that the SG handling was there to support crypto accelerators, which ""almost nobody"" has; the crypto developers think that having support for SG lists is a mistake, since there are so few accelerators. Hellwig said that there are a few accelerator drivers that do use SG lists, but that many crypto developers would like to remove that functionality; unfortunately, he said, the crypto maintainer wants to replace it with an even more complex interface. 

The new API ""looks really good and I can't wait to actually use it"", Chuck Lever said. He noted that he is the maintainer of a consumer of the ""RDMA rw interface"" and asked Hellwig whether that would be changed to use the new API. Hellwig said that it definitely should be, but he did not have the time or the hardware to do so; he was willing to help anyone who wanted to work on it and thought it should just be a few days worth of effort. 

Luis Chamberlain asked if Romanovsky could describe ""to the best of your ability"" what Murphy's objection to the patch set is. After a long pause, Romanovsky said: ""Honestly, I can't""; he said that he had tried to find ""any rationale, any action item that we can take"" from the responses, but was unable to. Hellwig said there were ""a lot of minor nitpicks"" that he was not sure were true; if they are, they need to be described better. Murphy has an overall objection that ""it pushes too much low-level knowledge into the consumers of the DMA API"", Hellwig said. But that is the point of the new API, Gunthorpe said; Hellwig concurred, noting that Murphy's objection was ""at least semi-valid"" though he did not agree with it. 

So what help were the developers asking for, Williams asked; ""is it just 'flood the zone with use cases', is it 'provide a wrapper API'?"". Hellwig said that the work on adding support to various users has been adding wrappers to make it easier to use and to hide the low-level details from the drivers; this work is meant to be a building block to allow subsystems to use their specific data structures for DMA, not to make the drivers individually handle those details. 

Gunthorpe agreed that having people post their use cases and their interest in using the feature would be useful, both for Murphy and for Marek Szyprowski, who is another DMA maintainer. No one is suggesting that Murphy's complaints should be ignored, he said, but there is a strong push for the changes from various users of the DMA interface. Maybe seeing additional use cases will encourage Murphy to spend more time on it. Williams thought that was a good path and recommended that ""everybody chime in, but be nice about it"". 

As of early May, Szyprowski had [merged the patches](<https://lwn.net/ml/all/2e89d7e2-9146-46c7-86b0-8023483e5e07@samsung.com/>) into his dma-mapping-next branch, presumably with an eye toward getting the new API into the 6.16 kernel.
