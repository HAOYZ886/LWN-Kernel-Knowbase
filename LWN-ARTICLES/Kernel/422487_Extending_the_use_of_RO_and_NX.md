---
title: Extending the use of RO and NX
url: https://lwn.net/Articles/422487/
date: "January 12, 2011"
category: "Security-Security technologies"
author: "By Jake Edge January 12, 2011"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
January 12, 2011

Pages of memory that are managed by the kernel are governed by access control flags that are somewhat analogous to the permissions which are applied to files. Those flags govern whether the page can be written to and whether its contents can be executed. Both attributes are useful to restrict what can happen to those pages in the presence of programming errors or security attacks. A pair of patches that were merged in the current merge window will further extend the usage of these flags for the x86 architecture. 

The page access flags, unlike file permissions, are enforced by the memory management hardware. The flags of interest for these patches are "write" and "execute", both of which imply "read" access, so they are often specified as follows: RO+X (read-only and execute) or RW+NX (read-write and no-execute). By restricting the usage of these pages, the scope of security flaws can be reduced because, for example, a buffer overflow in an NX page will not be directly useful for code execution. 

The memory that is used by the kernel to hold its read-only data (i.e. the `.rodata` segment) has been able to be marked read-only since 2.6.16 in early 2006, depending on the setting of `CONFIG_DEBUG_RODATA`. In 2.6.25, the kernel `.rodata` segment was additionally marked NX (i.e. no-execute), but only for the x86_64 architecture. A [patch](<https://lwn.net/Articles/342266/>) that was originally created for 2.6.30 (for both the 32 and 64-bit x86 architectures) expanded the use of NX for all kernel data pages, including read-write sections for initialized data and BSS. 

That patch was created by Siarhei Liakh and Xuxian Jiang but had fallen by the wayside after causing some [boot crashes](<http://lkml.indiana.edu/hypermail/linux/kernel/1011.1/00013.html>) on one of Ingo Molnar's test systems. When Kees Cook brought up the idea of doing better page access protection of the kernel's memory, Molnar [remembered](<http://lkml.indiana.edu/hypermail/linux/kernel/1011.1/01290.html>) that Matthieu Castet had ""dusted off those patches and submitted two of them"", back in August. After a few iterations, Molnar pulled them into the -tip tree, and Linus Torvalds pulled that for the mainline in the current 2.6.38 merge window. 

The revised [patch](<https://lwn.net/Articles/422541/>) itself is fairly straightforward. If `CONFIG_DEBUG_RODATA` is set, various sections of the kernel (`.text` and `.rodata`) are page aligned for both their start and end addresses. The NX bit is set for all pages from the end of the `.text` (i.e. code) section to the `_end` address that marks the end of the kernel's data section. 

There were two other pieces of the puzzle addressed in the patch, the first of which was presumably the cause of the boot crashes that Molnar had with the earlier patch. Some older systems that use PCI BIOS require that some pages in the 640K-1M region be executable. There are also some ISA mappings that require read-write access to that region. Rather than try to work all of that out, and potentially run afoul of buggy hardware, the patch just sets pages in that region to be RW+X on systems where PCI BIOS is used. The second change simply modifies `free_init_pages()` to turn on NX for any pages that are freed that way, so that those pages have to be explicitly allowed to store executable code when they are reused. 

A related [patch](<https://lwn.net/Articles/422540/>) adds read-only and no-execute flags to the pages used by kernel modules. It came from the same developers, and seems to have been dropped from -tip along with the NX patch. And, like the other patch, Castet pushed it the last bit to finally get it included in the mainline. 

The patch splits the module_core and module_init regions into three parts: code, read-only data, and read-write data. Each of those parts is page aligned and the page access permissions are set just before `load_module()` returns. For the code pieces, RO+X are set, while the data parts get NX and either RO or RW depending on the type of data. These changes are all governed by the setting of `CONFIG_DEBUG_SET_MODULE_RONX`. 

Beyond setting the page access control flags at module load time, the kernel must also reset those flags to RW+NX when the module is unloaded. In addition, the module_init region is freed after initialization is completed and its pages need to be put back to RW+NX. There is one further wrinkle: Ftrace needs to be able to modify the code in modules to enable tracepoints, so the patch provides a means for all module text pages to be set RW while Ftrace is making those changes, and then to set them back to RO afterward. 

Marking the kernel module pages as RO and/or NX is important not only because it is consistent with how the rest of the kernel pages are handled, but also because it makes other kernel protection efforts actually work for modules. For example, there has been an effort to declare structures of function pointers as `const`, so that exploits cannot change the pointers for their own nefarious purposes, but that only works if the `.rodata` pages are actually marked RO. 

The main cost of these patches is some bits of wasted memory from page aligning the various sections. Since that cost is probably not significant for any but the most resource-constrained embedded systems, it would make sense for `CONFIG_DEBUG_RODATA` and `CONFIG_DEBUG_SET_MODULE_RONX` to be turned on for most distributions--or to default to "on", though that is generally frowned upon by Torvalds and others. 

The fact that these patches have been around for a while, but never quite made the jump into the mainline is unfortunate. There is no real person or group that is currently shepherding core kernel security patches along, though Cook and Dan Rosenberg have recently been [making an effort](<https://lwn.net/Articles/415603/>) to push these kinds of changes. Cook's query helped resurrect both of these patches; they might have languished far longer without that interest. 

It is also worth noting that much or all of the protections embodied in these patches have long been available in the grsecurity/PaX kernels. While no wholesale import of the features from those kernels is ever going to happen, piecemeal patches that implement ""sane"" ([at least in Torvalds's eyes](<https://lwn.net/Articles/313621/>)) features can be adopted. That should lead to better kernel security, which is something that is certainly worth shooting for.
