---
title: Using large folios for text areas
url: https://lwn.net/Articles/1016416/
date: "April 8, 2025"
category: "Memory management-Huge pages"
author: "By Jonathan Corbet April 8, 2025 LSFMM+BPF"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
April 8, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

Quite a bit of work has been done in recent years to allow the kernel to make more use of large folios. That progress has not yet reached the handling of text (executable code) areas, though. During the memory-management track of the 2025 Linux Storage, Filesystem, Memory-Management, and BPF Summit, Ryan Roberts ran a session on how that situation might be improved. It would be a relatively small and contained operation, but can give a measurable performance improvement. 

[![\[Ryan Roberts\]](https://static.lwn.net/images/conf/2025/lsfmm/RyanRoberts-sm.png)](<https://lwn.net/Articles/1016423/>) Roberts began by saying that his objective is to make a kernel built for a 4KB page size to perform as well as one built for 64KB pages, at least when it comes to the handling of text. By mapping larger folios by default, he said, the kernel could take advantage of translation lookaside buffer (TLB) coalescing on newer processors and reduce memory-management overhead. The kernel's ability to use large folios has improved considerably; both anonymous memory and the page cache can work with them now. 

Text pages are managed through the page cache too, but the way they are accessed tends to prevent the use of large folios. When a file is being read from user space, the kernel's readahead system will detect sequential access and allocate large folios as data is read into the page cache. Execution tends not to be sequential, though; instead, it bounces randomly around the text section. As a result, sequential access is not detected, and the kernel, seeing random access, sticks with smaller folios. But, Roberts said, large (64KB) folios could be used for text without significantly increasing memory consumption. 

David Hildenbrand said that various people have tried using larger folios for text, but have not gotten as much of a performance improvement as had been expected. He wondered what sort of improvement Roberts expected from this kind of change. Roberts answered that he would get to the results, but that the short answer was that it depends on the workload, with some workloads seeing big improvements. 

For file readahead, Roberts continued, the kernel will bring in some data synchronously, then speculatively start an asynchronous read further ahead in the file. If that data is eventually used, the readahead size (and the size of folios used for that data) will be increased. For text, though, it is unusual for that asynchronous area to be accessed quickly, so everything ends up in small folios. A better approach, he said, would be to say that the asynchronous readahead just is not useful for text areas. Instead, the kernel could simply read the 64KB folio around the fault, without the speculative read beyond that folio. Then, he said, most text would end up in larger folios, which would be mapped together in the fault handler; that, in turn, makes it easy to set the page-table-entry bit needed for TLB coalescing on Arm systems. 

A participant asked whether it would be better to just use large folios unconditionally throughout the page cache; Roberts answered that the readahead code gets to that point anyway when large folios appear to make sense. Shakeel Butt pointed out that a lot of applications are built with their binaries organized into sections; perhaps there would be some useful hints there? Matthew Wilcox said that the compilers don't leave that information in the resulting binary, so those hints are not really available. 

Wilcox went on to say that he had learned a lot about the readahead code from the presentation — leading Roberts to interject that Wilcox had _written_ the readahead code. Wilcox said that the proposal makes sense in general, that the kernel should use 64KB pages for text. The readahead code, he said, is optimized for data, but text does not behave like data. 

Roberts moved on to the performance results, making it clear that he was only showing tests that improve with the new behavior. There were no performance regressions, though, just some workloads that did not show any difference. Overall, he said, most workloads saw a 4-8% performance improvement; that is less than the 12% that comes from going to a 64KB page size overall, but still worthwhile. 

He offered a few options for how this feature could be controlled, with the first being that each architecture would provide a preferred folio size for text mappings. The readahead code would gain a special case for memory areas that are mapped with execute permission; it would just perform a 64KB synchronous read in that case. If any of the pages in that folio are already in the page cache, though, then smaller reads would be performed. This solution would be entirely contained within the kernel, he said. He [posted an implementation](<https://lwn.net/ml/all/20240215154059.2863126-1-ryan.roberts@arm.com/>) of this option in February 2024, but received an objection that architectures should not be setting the folio size, so this work went cold. 

The second option would be to add a sysfs knob to allow the administrator to set the preferred folio size for text. He expressed a lack of enthusiasm for more knobs, though. The third option would be for the dynamic linker to make this decision at load time; a [`process_madvise()`](<https://man7.org/linux/man-pages/man2/process_madvise.2.html>) call could be made to inform the kernel of its decision. This moves the responsibility to user space and, he said, would create a new ABI, so he thought that this option was best avoided. 

Hildenbrand asked whether the `khugepaged` kernel thread could assemble larger folios after the fact; Wilcox said that it can do that now, but it tends not to run when it would be most useful. It would be far better, he said, to create the larger folios from the beginning. Hildenbrand asked if it would make sense to use a larger size, perhaps even 2MB, but Wilcox said that most executable segments are smaller than that. 

The fourth option, Roberts said, was not the right solution, but it had been raised on the list: the filesystem holding an executable could set the folio size. It could use the same infrastructure as the recently added large-block-size support. But, he said, there is no way for a filesystem to know which files should receive this treatment, or what a reasonable value would be. A filesystem-set size would also apply to the entire file, not just the text segments with it. As an extra bonus, he said, this option would have to be implemented in every filesystem separately. 

The session closed with a suggestion from Wilcox that the first option should be implemented. The others, he said, can always be added later if they seem to make sense. Roberts has subsequently [posted a new version](<https://lwn.net/ml/all/20250327160700.1147155-1-ryan.roberts@arm.com/>) of this work.
