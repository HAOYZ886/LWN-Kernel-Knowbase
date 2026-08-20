---
title: API changes under consideration
url: https://lwn.net/Articles/99120/
date: "August 24, 2004"
category: "copy  user-Proposed return value change; io remap page range"
author: 
---

There are two relatively significant API changes which are currently being tossed around for possible inclusion. Forewarned is forearmed, and all that, so here's a quick summary of what is being looked at. 

[2.6.8.1-mm4](<https://lwn.net/Articles/98923/>) included [a patch](<ftp://ftp.kernel.org/pub/linux/kernel/people/akpm/patches/2.6/2.6.8.1/2.6.8.1-mm4/broken-out/usercopy-return-EFAULT.patch>) which changes how `copy_to_user()` and `copy_from_user()` return a failure status. These functions have, for a long time, returned the number of bytes which they failed to copy to or from user space. This interface differs from what kernel programmers normally expect, and has caused confusion and bugs many times in the past. As [David Miller put it](<https://lwn.net/Articles/99124/>): 

People who are experts and work every day on their platform get this stuff wrong, myself included. This means we are too dumb to debug this code, according to The Practice of Programming :-) 

Rusty Russell also [expressed his opinion](<http://lwn.net/2002/0523/a/beartrap.php3>) on the `copy_*_user()` interface, as only Rusty can, a couple of years ago. 

Andrew Morton has decided that, perhaps, the time has come to fix the interface. In 2.6.8.1-mm4, the copy functions return the usual negative error code when things fail - at least, on the i386 platform. The change is overtly experimental, ""It's a see-what-breaks thing."" So far, reports of breakage are relatively scarce. 

On the other front, consider `remap_page_range()`. This function is prototyped as: 

```
int remap_page_range(struct vm_area_struct *vma, unsigned long virt,
                             unsigned long phys, unsigned long size, 
                             pgprot_t prot);
```

Its primary use is mapping memory found on I/O controllers into the virtual address space of a process. This function is accompanied by `io_remap_page_range()`, which is more explicitly intended for I/O areas. On almost every architecture, `io_remap_page_range()` is simply another name for `remap_page_range()`, but the SPARC architecture is different; it can make use of that architecture's I/O space to do things more efficiently. 

Paul Jackson recently [noticed](<https://lwn.net/Articles/99128/>) another difference: the SPARC versions of `io_remap_page_range()` have six arguments, while everybody else has only five. Needless to say, this is a curious discrepancy; it also makes it hard to write platform-independent code which uses `io_remap_page_range()`. 

The extra argument on the SPARC architecture is an integer "space" value; what it really is for, it turns out, is to specify the "I/O space" into which the pages are to be mapped. It is a response to a problem with the `remap_page_range()` interface: the physical address which is to be the target of the mapping is typed as an `unsigned long`. So a target address which requires more than 32 bits cannot be specified on 32-bit systems. SPARC I/O space addresses are above the 32-bit range. So the extra argument is required on the SPARC simply to provide the upper 32 bits for the physical address. 

Various options for smoothing out the difference were considered. In the end, the idea that seems to be winning is to change the `remap_page_range()` API slightly: instead of passing the target address as an address, that value should be expressed as a page frame number. That change gets rid of the 12 address bits used for the offset within the page (which are unused in `remap_page_range()` since that function deals in whole pages) and lets them be used for additional high-end bits, effectively extending the address range to 44 bits - which is enough. 

William Lee Irwin has put together [a patch](<https://lwn.net/Articles/99129/>) which implements this change for most architectures. Since the change breaks every caller of `remap_page_range()`, the patch touches a lot of files. Should the patch ever be merged, externally-maintained drivers will have to be fixed as well. This transition will not be helped by the fact that the compiler will not be able to detect unfixed code.
