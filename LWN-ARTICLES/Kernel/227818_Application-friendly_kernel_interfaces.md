---
title: "Application-friendly kernel interfaces"
url: https://lwn.net/Articles/227818/
date: "March 26, 2007"
category: "Development model-User-space ABI; Huge pages; User-space API"
author: 
---

The "hugetlb" feature of the kernel allows applications to create and use "huge" pages in memory. These pages use a special page table mode which allows a single page table entry to provide the translation for up to 16MB of contiguous memory (on some architectures). The advantage to doing things this way is that references to the entire huge page only take up one slot in the translation lookaside buffer (TLB), and that can have good effects on performance. 

Access to huge pages is through the hugetlbfs filesystem. Hugetlbfs is a virtual filesystem much like tmpfs, but with a twist: mappings of files within the filesystem use huge pages. It's not possible to do normal reads and writes from this filesystem, but it _is_ possible to create a file, extend it, and use `mmap()` to map it into virtual memory. This interface gets the job done, but it's evidently a little too involved for some application programmers. 

To make life simpler, Ken Chen has proposed [`/dev/hugetlb`](<https://lwn.net/Articles/227819/>). This device is much like `/dev/zero`, except that it uses huge pages. Applications can simply open the device and use `mmap()` to create as much huge-paged anonymous memory as they need. The patch is simple and seemingly uncontroversial; Andrew Morton did [note](<https://lwn.net/Articles/227824/>), though: 

afaict the whole reason for this work is to provide a quick-n-easy way to get private mappings of hugetlb pages. With the emphasis on quick-n-easy. 

We can do the same with hugetlbfs, but that involves (horror) "fuss". 

The way to avoid "fuss" is of course to do it once, do it properly then stick it in a library which everyone uses. 

He goes on to observe, however, that getting yet another library distributed widely can be a difficult task - to the point that it's easier to just add more functionality within the kernel itself. He concludes: ""This comes up regularly, and it's pretty sad."" 

In [a separate message](<https://lwn.net/Articles/227825/>), Andrew talked about how kernel interfaces should be designed in general: 

The fact that a kernel interface is "hard to use" really shouldn't be an issue for us, because that hardness can be addressed in libraries. Kernel interfaces should be good, and complete, and maintainable, and etcetera. If that means that they end up hard to use, well, that's not necessarily a bad thing. I'm not sure that in all cases we want to be optimising for ease-of-use just because libraries-are-hard. 

In many cases, the C library fills this role by providing a more application-friendly interface to kernel calls. But there are limits to how much code even the glibc developers want to stuff into the library, and things like a friendlier huge page interface may be on the wrong side of the line. A separate library for developers trying to do obscure and advanced things with the kernel might be the right solution. 

The right solution, Andrew suggests, is to have a user-space API library which is maintained as part of the kernel itself. That would keep oversight over the API and help to ensure that the library is maintained into the future while minimizing the amount of code which goes into the kernel solely for the purpose of creating friendlier interfaces. Somebody would have to step up to create and maintain that library, though; as of this writing, volunteers are in short supply.
