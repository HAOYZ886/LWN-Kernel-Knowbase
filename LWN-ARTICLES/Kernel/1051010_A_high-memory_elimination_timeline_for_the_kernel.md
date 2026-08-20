---
title: "A high-memory elimination timeline for the kernel"
url: https://lwn.net/Articles/1051010/
date: "December 23, 2025"
category: "Architectures-32-bit systems"
author: "By Jonathan Corbet December 23, 2025 LPC"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
December 23, 2025

* * *

[LPC](<https://lwn.net/Archives/ConferenceIndex/#Linux_Plumbers_Conference-2025>)

Arnd Bergmann began his [2025 Linux Plumbers Conference](<https://lpc.events/event/19/>) session on the future of 32-bit support in the Linux kernel by saying that it was to be a followup to [his September talk](<https://lwn.net/Articles/1035727/>) on the same topic. The focus this time, though, was on the kernel's "high memory" abstraction, and when it could be removed. It seems that the kernel community will need to support 32-bit systems for some time yet, even if it might be possible to remove some functionality, including support for large amounts of memory on those systems, more quickly. 

#### The high-memory problem

High memory, he began, is needed to support 32-bit systems with more than 768MB of installed physical memory with the default kernel configuration; it can enable the use of up to 16GB of physical memory. The high-memory abstraction, though, is a maintenance burden and ""needs to die"". Interested readers can learn more about high memory, why it is necessary, and how it works in [this article](<https://lwn.net/Articles/813201/>). 

[![\[Arnd Bergmann\]](https://static.lwn.net/images/conf/2025/ossjp/ArndBergmann-sm.png)](<https://lwn.net/Articles/1051013/>) A 32-bit system has a 4GB virtual address space (the physical address space is larger), which is split between user and kernel space. There are various kernel-configuration options that can change where that split happens; the most common configuration is `VMSPLIT_3G`, which allocates the bottom 3GB of the address space to user space, while reserving 1GB for the kernel. The problem is that the kernel's "linear map" (or "direct map"), which maps all of physical memory, must fit in the kernel's part of the address space; with `VMSPLIT_3G`, that limits the kernel to directly mapping 768MB of physical memory. Any memory beyond that is managed as high memory, which must be explicitly mapped before every use (and unmapped afterward). 

Configuration options that increase the size of kernel space, and thus the size of the linear mapping, exist, but they do so at the cost of reducing the address space available to applications. For example, the `VMSPLIT_2G` options support up to nearly 2GB of physical memory without using high memory, but limit the virtual address space to 2GB. 

There are a number of reasons to want to get rid of high memory, Bergmann said. Embedded developers dislike it because it tends to be the source of regressions on updates and complicates the code significantly. While 32-bit CPUs are still used for embedded applications, they normally do not have large amounts of memory installed, so high memory is not particularly helpful there. The memory-management maintainers dislike high memory for the same reasons; they would also like to see it disappear before anybody starts to think that 64-bit high memory might be a good idea. 

There are reasons to keep high memory too, of course. Removing it increases the chances of exposing driver bugs and can force changes to user-space code. Any 32-bit system with more than 2GB of installed memory cannot actually use that memory without the high-memory abstraction. Even smaller amounts of memory cannot be supported without reducing the size of the user-space address space as described above, which would break applications that use a lot of virtual memory. If high memory were removed from the kernel, there would be no hope of supporting 32-bit systems with more than 4GB of memory; even 2GB systems would suffer significant limitations. There would be no impact at all, instead, on systems with less than 1GB installed. 

There are still 32-bit systems being made, but almost all of them have 1GB or less of installed memory. The only reason to use a 32-bit CPU in a new system, he said, is extreme cost sensitivity, so there is unlikely to be a budget for larger amounts of memory. Anything with more than 1GB is thus almost certainly an older system. The 1GB systems are relatively easy to support, except that some of them have discontiguous memory, which makes it impossible to map into the kernel's linear mapping. There are workarounds, but some of them might require user-space changes. 

The 2GB case is becoming rare, he said, but people do still have these systems, so support for them may have to be maintained as long as support for 1GB systems. The 2GB `VMSPLIT` option may work for some, but it results in a 1.75GB virtual address space, which is too small to run Firefox. David Hildenbrand asked whether "support" means that new kernels have to work on these systems; the answer was "yes", at least for systems where users still want to be able to update. 

Then, there are systems with more than 2GB of memory. These can be old (pre-2007) x86 laptops or Arm Chromebooks from 2012 or 2013. There are also evidently in-flight entertainment systems, fire-alarm systems, and digital signs that fall into this category. These are systems where the cost of the board is a tiny part of the total cost, he said, so the manufacturers go ahead and put in more memory. Jason Gunthorpe asked whether people were really upgrading these systems to current kernels; Bergmann said they were, and Gunthorpe responded that he was shocked to hear that. 

There are a number of known 32-bit systems with more than 4GB of memory; Bergmann said they were all ""dead systems"". They include Amazon Annapurna Alpine boxes, Calxeda Midway systems, HiSilicon HiP04 systems, among others. Evidently there are also SPARC systems with up to 2GB of memory that are still getting updates. 

#### Proposals

So what is to be done about high memory? Bergmann said that he had held out a lot of hope for a `VMSPLIT_4G` option, which would separate kernel and user space entirely, giving the full 4GB to each. That would allow systems with up to nearly 4GB of physical memory to be supported without high memory and would have solved a lot of problems, but this option has never been pushed to the point where it actually works, and nobody is funding that work now. So this option will probably never happen, he said, but it also probably will not be necessary. 

What may be needed is an option called "densemem", which uses some mapping trickery to close up holes in the physical address space. Densemem is needed to [replace the SPARSEMEM model](<https://lwn.net/Articles/974517/>) in any case, and it could enable support for systems with up to 2GB without high memory. This option would reduce the address space available to user space, though. It also is not working yet; Bergmann is looking for developers who want to help finish it. 

Another thing that is likely to happen is "reduced-feature high memory", where high-memory support would be dropped from one subsystem at a time. That would reduce the complexity of the system, and would also reduce the impact of an eventual high-memory removal. So, for example, support might be removed for page tables, DMA buffers, filesystem metadata, and more in high memory. Other users, such as file-backed and anonymous memory mappings, would need to continue to use high memory for now. 

Years ago, low memory was a relatively scarce resource, so developers were advised to provide the `__GFP_HIGHMEM` flag (indicating that the request could be satisfied from high memory) whenever possible. Bergmann suggested changing that policy so that memory allocations would only include `__GFP_HIGHMEM` where high memory is truly required. He said that it might be possible to start phasing out 32-bit desktop use cases in particular, though he acknowledged that such a move might be controversial. Gunthorpe wanted to confirm that no such desktops are being made anymore; Bergmann said that is the case. The users of 32-bit desktop systems are hobbyists and people who actively seek out old hardware. Dave Hansen described those users as ""a vocal minority"", and suggested that they might be better off supporting a computer-history museum. 

Eventually, Bergmann said, high memory could be placed behind the `CONFIG_EXPERT` configuration option, making it inaccessible to many users. 

Hildenbrand asked what the impact on maintenance would be for the partial removals that had been proposed; Bergman said that it was small, but it would help with the following removal stages. Gunthorpe worried about the possibility of exposing driver bugs, and that removing high-memory support from them may not be worth the trouble. He suggested targeting the areas where high memory creates a lot of complexity instead. Matthew Wilcox said that removal could break kernel code that has to map large folios; Bergmann suggested making large-folio support incompatible with high memory. Hansen said that the `__GFP_HIGHMEM` flag could simply be removed for large-folio allocations, but Bergmann said that would break the page cache. Gunthorpe said that, when high-memory support is removed, systems that need it for the page cache simply will not work anymore. 

Bergmann went on, suggesting that a separate configuration option could be created for each high-memory user. That would allow the creation of statistics for each usage. High-memory support for the page cache, he said, would need to be retained for at least five more years. 

#### The timeline

He concluded with a suggested timeline for the future of 32-bit support; it looked like this: 

  * The high-memory feature-set reduction would begin in 2026. 
  * Also in 2026, the `VMSPLIT_2G_OPT` configuration would become the default for Arm, PowerPC, and x86 systems. 
  * In 2027, high-memory support could be removed entirely for lesser-used architectures like Arc, Microblaze, MIPS, SPARC, and xtensa. 
  * With luck, 2027 will also see the addition of densemem support for Armv7 systems. 
  * In 203x, support for the page cache in high memory would go away. 
  * In 204x, support for the last 32-bit architectures would be removed. 

Hansen reacted by saying that many of the removals could be done more quickly. He also suggested a configuration option to only use low memory, even on systems with high-memory support, until forced by an out-of-memory situation to use the high memory too. Will Deacon said there might yet be hope for `VMSPLIT_4G`, which may become more attractive as high-memory support is removed. He asked if it would be useful if somebody completed the work; Bergmann answered that it would, but whether that work will be done is unclear. 

After the conference, Bergmann posted [a patch series](<https://lwn.net/ml/all/20251219161559.556737-1-arnd@kernel.org>) implementing parts of his 2026 timeline items. 

The [slides](<https://lpc.events/event/19/contributions/2261/attachments/1823/3963/Highmem%20deprecation%20planning-1.pdf>) and [video](<http://www.youtube.com/watch?v=ka0AJvH6uPE>) of this talk are available. 

[Thanks to the Linux Foundation, LWN's travel sponsor, for supporting my travel to this event.]
