---
title: Toward better handling of major page faults
url: https://lwn.net/Articles/1073071/
date: "May 22, 2026"
category: "Memory management-Scalability; Memory management-mmap sem"
author: "By Jonathan Corbet May 22, 2026 LSFMM+BPF"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
May 22, 2026

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2026/>)

A major page fault occurs when a process attempts to access a page that is not currently present in RAM; satisfying such faults usually involves I/O, and can thus take some time. When many threads sharing an address space are generating page faults, the result can be significant lock contention while that I/O takes place. During the memory-management track at the 2026 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>), Barry Song led a session to try, yet again, to find an enduring solution to this problem. 

Song began by saying that [per-VMA locks](<https://lwn.net/Articles/906852/>) had been introduced a few years back; that work moved much of the page-fault-handling work to a lock at the virtual-memory-area (VMA) level, in an attempt to relieve pressure on the process-level `mmap_lock`. But, when satisfying the fault requires initiating I/O, the kernel will release the per-VMA lock then retry the handling of the fault, with `mmap_lock` held, once the I/O completes. That can create significant `mmap_lock` contention, causing threads to stall. He wanted to find ways of reducing that contention, and had a few options to consider. 

The first is to simply retry the handling of the fault using the VMA lock rather than `mmap_lock` after I/O completion; he has posted [a patch series](<https://lwn.net/ml/linux-mm/20251127011438.6918-1-21cnbao@gmail.com/>) implementing this idea. That would introduce even more complexity into the fault-handling path, though, he said. 

An alternative would be to completely remove the retry code and, instead, simply hold the VMA lock while waiting for I/O. Lorenzo Stoakes worried that this approach, too, would add complexity. Shakeel Butt asked about how bad the additional complexity would be; Matthew Wilcox answered that it would not be that much worse, but that the fault-handling code is already too complex now. He said there might be a possible third option: apply Song's change to retry under the VMA lock, but only for anonymous pages, where the change is relatively simple. 

Ryan Roberts said that the retry flag (the `VM_FAULT_RETRY` value returned when fault handling must be retried) is covering too many cases. It is used for compatibility with code that is not able to deal with the VMA lock and, as a result, retry has to use `mmap_lock`. Suren Baghdasaryan said that retries are called for when an operation cannot be done under the VMA lock — at least, not at the current time. There might be a place for a separate flag to call for a retry under the VMA lock. Butt asked whether the contention stalls Song had observed were associated with anonymous or file-backed pages; the answer was that the problem is mostly seen with the latter. 

Song returned to the option of removing the retry code entirely. He said it would be possible, but has the potential to create priority-inversion problems. Threads running within an Android app have different priorities; in the wrong scenario, one thread holding `mmap_lock` could block the high-priority user-interface thread, causing visible stalls. Wilcox said that the real problem is threads waiting for I/O with the VMA locked, but Song said that the problem comes up even if the high-priority thread is not accessing the same VMA. After some discussion on whether the priority-inversion scenario was a real problem, the consensus seemed to be that it indeed is. 

Song concluded with a few other discussion points, the first of which was whether it makes sense to use different approaches for anonymous and file-backed pages. In the case of anonymous pages, the kernel can allow page-fault handling and other VMA changes (an [`mprotect()`](<https://man7.org/linux/man-pages/man2/mprotect.2.html>) call, for example) to happen concurrently. The file-backed side might benefit more from removing the retry logic entirely once the priority-inversion problem has been fully understood and avoided. Then, he said, there are cases where multiple threads are faulting in the same sets of pages; rather than contend on the `mmap_lock` and folio locks, the handler could check the up-to-date status of the folio. If it is up to date, then somebody else has already performed the I/O to handle the fault, so the retry can be avoided. 

His final question was whether the kernel should retry fault handling under the VMA lock by default and only fall back to the `mmap_lock` in cases where it is known to be needed. Baghdasaryan repeated the idea of adding a new retry flag to indicate that the VMA lock should be held. The session ended with a suggestion from Vlastimil Babka that Song should try the various options to see how they work. 

Song has posted [his slides](</images/conf/2026/lsfmm/song-slides.pdf>) from this session.
