---
title: "LZ4: vendoring in the kernel"
url: https://lwn.net/Articles/713175/
date: "February 1, 2017"
category: Development model
author: "By Jonathan Corbet February 1, 2017"
---

By **Jonathan Corbet**  
February 1, 2017

In [his 2017 linux.conf.au talk](<https://lwn.net/Articles/712376/>), Robert Lefkowitz talked about the process of "vendoring" — the copying of code from other projects into one's own repository rather than accepting a dependency on those projects. Vendoring is common in commercial projects; Android does a lot of it, for example, and Lefkowitz suggested that the practice should become more common elsewhere as well. Vendoring is not unknown elsewhere and is even done in the kernel, as a current patch set shows. 

The [LZ4 compression algorithm](<https://github.com/lz4/lz4>) claims to be "extremely fast", especially on the decompression side. The project claims benchmark results showing LZ4 beating LZO decompression by a factor of four and zlib by nearly an order of magnitude. It is a lossless algorithm, so it is suitable for compressing data that must be recoverable in exactly its original form. Recent releases have added a "fast" mode that allows callers to control the trade-off between speed and the amount of compression applied. 

One can imagine how this kind of fast compression would be useful to have in the kernel. And indeed, the kernel has had LZ4 capability since the 3.11 release in 2013. It was added by Chanho Min, who grabbed the r90 release from the LZ4 repository and stuffed it into the kernel under `lib/lz4`. A quick grep shows that it is currently used in the crypto layer, in the pstore subsystem, and in the squashfs filesystem. There are other places in the kernel that use compression, but they are not using LZ4 currently. 

One of the advantages of copying the code into your own repository is that you are no longer dependent on an external dependency. Lefkowitz thought that independence was so valuable that he recommended copying for any dependency with less than about 35 million lines. In the kernel's case, there is an especially strong case against external dependencies: the kernel must be built as a standalone program using its complicated set of linker rules. It is probably possible to tweak the kernel's build system to allow it to link against externally supplied libraries, but one can imagine that there would be a fair amount of opposition to any such move. Kernel developers want to know exactly what is going into the end product. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

The downside of vendoring, of course, is that you then lose out on all of the enhancements made in the original project. The LZ4 developers have [made a number of releases](<https://github.com/lz4/lz4/blob/dev/NEWS>) since 2013; these have added numerous features, including the "LZ4 fast" mode. Some of the changes may have fixed bugs that, in the kernel, would constitute security vulnerabilities. None of those changes are in current kernels. 

Toward the beginning of the year, Sven Schmidt [posted a patch set](<https://lwn.net/Articles/713191/>) updating LZ4 to the project's 1.7.2 release. The motivation was a desire to use the LZ4 fast mode in the Lustre filesystem, but he made the reasonable assumption that other parts of the kernel might want to take advantage of the fast mode as well. The patches are a wholesale replacement of the existing LZ4 code; the work initially done by Min to turn the LZ4 library into a kernel module has been replicated. 

There do not appear to be any objections to upgrading the kernel's LZ4 implementation, but Greg Kroah-Hartman did [note](<https://lwn.net/Articles/713192/>) one potential problem and, in the process, highlighted one of the other hazards that go with vendoring. The existing in-kernel LZ4 implementation has not sat still since 2013; it has had a number of patches applied to it. Some of those were security fixes. When Schmidt replaced the LZ4 implementation, he replaced those fixes as well, potentially reintroducing problems that had already been fixed once. 

Once his attention was called to the issue, Schmidt agreed to look at the patches and make sure that his replacement does not bring the old bugs back. With luck, he will also get any relevant changes merged back upstream, though Willy Tarreau [suggested](<https://lwn.net/Articles/713194/>) that some of the fixes, at least, were specific to the kernel. If such changes exist, they are unlikely to make it upstream and will thus be something the kernel has to carry indefinitely. 

Making sure that the new LZ4 maintains the fixes applied to the old one is not a huge job; the number of patches is small. Happily, they exist as separate patches, rather than having been quietly folded into the source when LZ4 was initially added to the kernel. But it is a job that has to be remembered every time that somebody decides to update the kernel's LZ4 implementation. In this case, Kroah-Hartman noticed the problem, but the project cannot always count on his attentiveness to avoid regressions with future upgrades. 

Such upgrades will almost certainly happen sooner or later. The upstream LZ4 project is already up to 1.7.6 as of this writing; it has added a new high-compression mode and fixed some bugs since 1.7.2 was released. At some point, somebody working in the kernel space will want the enhancements being made upstream. 

The kernel has other copied subsystems like LZ4; they are mostly low-level compression and cryptographic code. Each one of these represents a sort of disconnect from the upstream project (in cases where there is still a functioning upstream project, at least). One could regard the highly modified kernels shipped in the mobile and embedded areas as being another example of the same thing; rather than upstream their code, these vendors simply copy it from one kernel to the next. 

There are solid reasons for vendoring, but also real costs associated with it. The prevalence of vendoring throughout our community suggests that we are still struggling to find the best ways to integrate software that is created by independent groups of developers, especially as the scale of our projects continues to increase. For now, we will just have to hope that, the next time somebody decides to update a library like LZ4 in the kernel, they will remember what the old fixes are and make sure they carry over to the new version.
