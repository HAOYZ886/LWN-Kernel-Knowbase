---
title: DMA and get_user_pages()
url: https://lwn.net/Articles/774411/
date: "December 12, 2018"
category: "Memory management-get user pages"
author: "By Jake Edge December 12, 2018 Linux Plumbers Conference"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
December 12, 2018

* * *

[Linux Plumbers Conference](<https://lwn.net/Archives/ConferenceByYear/#2018-Linux_Plumbers_Conference>)

In the RDMA microconference of the 2018 [Linux Plumbers Conference](<https://linuxplumbersconf.org/>) (LPC), John Hubbard, Dan Williams, and Matthew Wilcox led a discussion on the problems surrounding `get_user_pages()` (and friends) and the interaction with DMA. It is not the first time the topic has come up, there was also a [discussion](<https://lwn.net/Articles/753027/>) about it at the Linux Storage, Filesystem, and Memory-Management Summit back in April. In a nutshell, the problem is that multiple parts of the kernel think they have responsibility for the same chunk of memory, but they do not coordinate their activities; as might be guessed, mayhem can sometimes ensue. 

Hubbard began by laying out the goals of the session. The idea is to make sure everyone knows about the problem; even though it has been discussed in various mailing-list threads and such, everyone may not be up to speed on it. He was hoping to get a consensus on a long-term fix; he has put out a few RFCs, but the solutions have been a bit contentious. 

It is not that difficult to crash the kernel using a common code pattern, he said. An application pins pages in memory, which causes the kernel to use `get_user_pages()` or similar to map the user-space memory into the kernel's address space. It then sets up DMA for the pages. Though it was the _R_ DMA microconference, the problem exists for all DMA operations (for, say, a GPU or FPGA), not just remotely over the network, Hubbard said. The drivers will then mark the destination pages as dirty and release them with `put_page()` (or `release_pages()`). But if the pages are backed by a non-RAM-based filesystem, things can go awry. This pattern has been "known" to be reasonable since 2005 or so, but it is actually broken. 

[ ![\[John Hubbard & Dan Williams\]](https://static.lwn.net/images/2018/lpc-hubbard-williams-sm.jpg) ](<https://lwn.net/Articles/774494/>)

The underlying problem is that the page buffers can be stripped off while the page is pinned and undergoing I/O; the writeback code believes it has written the dirty data, thus the buffers are not needed, but then the driver marks the pages dirty again when the I/O completes. Once that happens, writeback may come along (again) expecting that the page buffers are still there. Essentially, the filesystem is unaware that a page can be marked dirty outside of its purview. There are two subsystems that do not agree on which is the maintainer of the dirty state. All of that means that DMA to file-backed memory is not truly supported by Linux today, he said. 

Hubbard went into some more detail on the crash, which can be seen in his [slides [PDF]](<https://linuxplumbersconf.org/event/2/contributions/126/attachments/136/168/LPC_2018_gup_dma.pdf>) that also reference a [lengthy email](<https://www.spinics.net/lists/linux-mm/msg142700.html>) from Jan Kara about the problem. Kara was listed as one of the session leads, but he could not make it to LPC. 

In answer to a question from an attendee, Hubbard said that the buffers are being freed because reclaim thinks it is going to get further than it does. It cannot progress because `get_user_pages()` has taken a reference to the pages. Reclaim has already released the buffers, but then has to back out. Williams added that if reclaim stays out of the way, everything works just fine; the problems occur when reclaim gets to the pages before the driver does. 

Mel Gorman said that filesystems fundamentally assume that `page_mkwrite()` will be called before a page can be marked dirty. So when reclaim comes along, the page may be dirty, but the filesystem thinks it is safe to get rid of the buffers. He noted that dirty is not really a binary state. 

Someone asked how big of a problem this is, does it occur every day, week, or month? Hubbard said it is reliably reproducible for him; he has advised customers to avoid doing DMA to file-backed memory. But it is rare enough that people write this kind of code, test it successfully, and then it fails when it is deployed in a more stressful scenario, an attendee said. Another suggested that the less reproducible a bug is, the more important it is to fix it. Hubbard agreed, saying he remembered a bug that took a month to reproduce, which meant it ended up taking him a year to fix it. Wilcox added that it definitely needs fixing as more and more systems are running into it. 

#### Proposal

Hubbard then proposed that `get_user_pages()` "make a note" in the page structure that effectively says "`get_user_pages()` was here". One problem is that `struct page` is full—a perennial problem. An idea that Wilcox had come up with is to use the LRU next and previous pointers in `struct page` for flags and a reference count to track this; the pages would be removed from the LRU list while they are pinned, Hubbard said. 

The way Hubbard is proposing to do this is to replace all of the affected `put_page()` and `release_pages()` calls with new `put_user_page*()` calls that will call `put_page()` after dealing with the tracking information and restoring the pages to the LRU list. His [RFC patch set](<https://lwn.net/ml/linux-fsdevel/20181110085041.10071-1-jhubbard@nvidia.com/>) converts the InfiniBand driver, but there are around 100 other places that need to be converted. Some of those will need a fair amount of analysis to determine how to convert them, he said. 

Williams cautioned that he is using the LRU pointers in DAX and that Jérôme Glisse is also using them in his heterogeneous memory management (HMM) patches. Wilcox admonished that he had freed up "a lot of space" in `struct page` for DAX and HMM; "there's going to be space for both of you". It is important to fix this, Hubbard said, since nothing stops users from doing it, as they have since 2005; "it just doesn't work", Williams added. 

After the first RFC was posted, Andrew Morton asked how Hubbard was going to ensure that all of the conversions have been done. There is a need to not only solve the problem, but to show that it has been solved. To that end, he wants to add an assertion into the code that will trigger if `put_user_pages()` has not been called when it should have been. He needs a bit in `struct page` to track that, however. Wilcox said that he had another bit Hubbard could use, especially if it was only a temporary use while the conversion was in progress; "I've been saving it for you John", Wilcox said with a grin. Williams cautioned that "the first bit is free"—to widespread laughter. 

Part of what he is trying to accomplish in the session is to raise awareness that this patch set is coming, Hubbard said. The RFC is just six patches that includes the InfiniBand conversion, which has been reviewed by Infiniband developers. But he wants to take some time to go into each subsystem and make sure that he understands how it is using the memory. For now, there is a dummy `put_user_page()` that simply does a `put_page()`. 

All of the work he had described so far just tracks the pages that have been pinned with `get_user_page()`. Once you have that information, you need to do something with it. Stopping the buffer removal is needed and can be done by holding off `try_to_unmap()` for those pages. But filesystem developers say that holding pages in that state is not a good idea, so Kara and others have said that the kernel could still allow writeback, but do it with bounce buffers that will be unaffected by whatever else is done with those pages. 

#### `revoke()`

Williams said that he wanted to talk about APIs for `revoke()`, which would help with these problems where an `mmap()` region is shared and being used for DMA. If another process wants to truncate or punch a hole in the file in the region where DMA is being done, "you are screwed", at least for DAX. 

But, as he has in the past, Glisse pointed out that there are some things that cannot be revoked. It will only work on devices that support it. Some drivers can stop the DMA, but others just pin the memory and have no way to stop the DMA and unpin the memory. Wilcox said that it is hardware dependent and, even for NVMe, which has a command to cancel outstanding I/O, most devices just do not implement it. That memory cannot be reused until the hardware says it won't DMA there anymore, he said. Williams suggested that the `revoke()` call should just wait for the device, but for some devices that wait might be forever, an attendee said. 

Everything that calls `get_user_pages()` needs to be audited, but Williams was concerned that some drivers don't really know whether their pages have come from `get_user_pages()` or not. Hubbard said that he had to pass some extra tracking information down in some of his under-development conversions. If he could permanently free up a bit in the page structure to track whether these were pages pinned with `get_user_pages()`, Wilcox wondered, would that remove the need to change all of the call sites, since `put_page()` could simply do the right thing? 

There was some thought that might work, but it didn't last long. There would be no way to distinguish other uses of `put_page()` from one that should do that "right thing". There would be a reference count, but there is no way to know that any given `put_page()` is the one from the driver that should clear the bit. So the call sites in the drivers still would need to change. It would help tracking these kinds of pages so that the extra information would not have to be passed down from upper layers, however. 

The performance numbers that accompanied the RFC patches were questioned by an attendee. While they showed little performance impact for the changes, the numbers were orders of magnitude lower than what these kinds of devices should be able to do. The concern is that the numbers are so low that they shouldn't even be compared to determine what impact, if any, the changes actually made. As can be seen in a [thread](<https://lwn.net/ml/linux-fsdevel/942cb823-9b18-69e7-84aa-557a68f9d7e9@talpey.com/>) from after LPC, there was a measurement problem; once it was resolved, though, the impact still is minimal. 

Returning to the subject of `revoke()`, Williams said that his goal was to get rid of the distinction between short-term and long-term DMA. That distinction was added so that DAX could simply fail any attempt to use its memory for long-term DMA. It is not just RDMA that does it; there are other devices, such as video-offload devices, that also pin memory forever. He suggested that maybe file leases could provide a model for handling the problem. if another process does a file truncate operation, the lease mechanism could provide a way to pull that memory away from the device. 

Jason Gunthorpe suggested simply failing the truncation operation when the memory is in use for DMA. Wilcox said that he had been advocating that for years, but that other kernel developers will not allow it. 

The pages belong to the filesystem, Boaz Harrosh said, so the mistake is in letting other parts of the kernel handle them in ways the filesystem cannot see. He suggested that DMA users take out a range lock for the part of the file that is being used for DMA, but Williams said that they cannot "rewrite the universe" to say that everyone must get a range lock. That will also have performance impacts that are unacceptable, Wilcox said. 

It is clear that there are no easy solutions, but the planned path seemed agreeable to most. Wilcox pointed out that by removing the pages from the LRU list in `get_user_pages()`, writeback would never find them, so the crash cannot occur. For DAX, things are not so rosy, but the RDMA developers seemed willing to try to handle being told the memory they had pinned for DMA was going away, at least in exceptional circumstances. It certainly seems like a topic that will come up again—possibly multiple times over the next few years. 

A [YouTube video](<https://www.youtube.com/watch?v=97mmajwTjpY&t=5290>) of the session is available. 

[I would like to thank LWN's travel sponsor, The Linux Foundation, for assistance in traveling to Vancouver for LPC.]
