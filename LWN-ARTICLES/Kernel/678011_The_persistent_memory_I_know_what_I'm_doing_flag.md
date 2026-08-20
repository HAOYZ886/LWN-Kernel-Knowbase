---
title: "The persistent memory \"I know what I'm doing\" flag"
url: https://lwn.net/Articles/678011/
date: "March 2, 2016"
category: "DAX; Memory management-Nonvolatile memory"
author: "By Jonathan Corbet March 2, 2016"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
March 2, 2016

As was described in [Neil Brown's article](<https://lwn.net/Articles/676737/>) last week, developers working on persistent memory appear to be converging on a solution for the `fsync()` system call. A working `fsync()` will enable applications to ensure that the data they have written is safely stored to persistent memory; importantly, applications that have been written correctly for POSIX filesystems in general will work correctly on persistent memory without the need to be aware of the difference. But some developers _want_ to write code that is specific to persistent memory as a way of maximizing performance. A patch catering to the needs of those developers inspired a lengthy conversation on how to best ensure that data written to persistent memory is not lost, and how development in this area should proceed in general. 

The problem with the emerging `fsync()` solution, according to Boaz Harrosh, is that it requires the kernel to maintain a radix tree of all pages that might have dirty lines of data in the CPU caches. If an application has been written with persistent memory in mind, though, it can avoid leaving data in the caches. That data can be explicitly flushed by the application or, as an alternative, [non-temporal writes](<https://lwn.net/Articles/255364/>) can be used to bypass the CPU caches entirely. If the application is using these techniques, Boaz said, there is no need for the kernel to flush cache lines for the relevant persistent memory, so it can avoid the wasted overhead of maintaining the radix tree. 

The kernel currently has no way of knowing that an application is taking care of its own cache-management needs, though. Fixing that is the goal of [this patch set](<https://lwn.net/Articles/676743/>) posted by Boaz in February. It adds a new flag for the `mmap()` system call named `MAP_PMEM_AWARE`. If an application maps a file stored in persistent memory with this flag, and the filesystem supports the [DAX](<https://lwn.net/Articles/610174/>) direct-access mechanism, the kernel can assume that the application will deal with cache management and, as a result, the kernel need not track pages with potentially dirty cache lines. Boaz claims considerably improved performance when running with this patch. 

#### Some concerns

It is fair to say that this patch was not universally acclaimed. There were a number of objections to providing this kind of functionality, the first of which being that an application that does its own cache management will still have to make calls to `fsync()` (or `msync()`) to ensure that its data is truly persistent. That is because this data does not stand alone; it is stored within a filesystem, and the application has no knowledge of whether there is any filesystem metadata that must also be flushed out to be sure that the data can be accessed. The only way to be sure that the metadata is consistent on disk is to call `fsync()`, just like applications dealing with data on more traditional storage media. 

In theory, an application can allocate and write an entire file, then call `fsync()` to get it all to persistent storage with the goal that, afterward, it can rewrite the data within the file without causing any further metadata changes (other than timestamps, which are not important for retrieving that data). But filesystems can be performing actions like data deduplication, delayed allocation, or, as Christoph Hellwig [pointed out](<https://lwn.net/Articles/678017/>), copy-on-write operations. So it is true that the only way to be sure that data is truly, safely persistent is to call `fsync()`; the `MAP_PMEM_AWARE` flag would not eliminate that requirement. 

Boaz [protested](<https://lwn.net/Articles/678018/>) that eliminating the need to call `fsync()` was never the purpose of the patch set. Instead, it aims to make those calls much faster; other overhead, especially associated with page faults in areas backed by persistent memory, would also be significantly reduced. Unfortunately, the worries about `MAP_PMEM_AWARE` didn't end there. 

For example, consider the interaction between applications using this flag and others that are not aware of persistent memory. Such applications (which might be something as simple as `mv` or a backup utility) may also create metadata changes needing flushing, and they may create dirty cache lines in the persistent-memory area that the "aware" application knows nothing about. Experience with direct I/O has shown that such interactions can be subtle, difficult to notice, and impossible to fix. 

Perhaps the biggest worry, though, is that application developers will rush out and proclaim that their code is "aware" without actually understanding everything they need to do to guarantee the integrity of their data. As Dave Chinner [put it](<https://lwn.net/Articles/678021/>): ""Almost any app developer that says they understand how filesystems provide data integrity is almost always completely wrong."" If the kernel provides these developers with an "I know what I'm doing" flag, the reasoning goes, they will soon write code that demonstrates the lack of that knowledge — to their users' detriment. 

One might just say that any such applications are buggy; they will either be fixed or replaced with something better. But, as Dave continued, he made it clear that he didn't see things happening that way. 

History tells us otherwise. Users always blame the filesystem first, and then app developers will refuse to fix their applications because it would either make their app slow or they think it's a filesystem problem to solve because they tested on some other filesystem and it didn't display that behaviour. The result is we end up working around such problems in the filesystem so that users don't end up losing data due to shit applications. 

The same will happen here - filesystems will end up ignoring this special "I know what I'm doing" flag because the vast majority of app developers don't know enough to even realise that they don't know what they are doing. 

That last point is key: filesystem developers, in their own defense, will end up ignoring this new flag because the alternative is to face the wrath of users who blame them for their lost data. The [ext4 data-loss wars](<https://lwn.net/Articles/322823/>) in 2009 have left some lasting scars; filesystem developers do not wish to find themselves in that position again. 

#### Data integrity first

Developers had one more reason to oppose this patch — one that had little to do with the specifics of the patch itself. DAX and its associated persistent-memory functionality are still new, and problems are still being found with them. Dave [made the claim](<https://lwn.net/Articles/678024/>) that the core problem of safely storing data via DAX has not yet been solved, so it is not appropriate to be looking at optimizations. For now, the focus has to be on making things reliable; after that, there will be time to look at where the performance issues lie and do some optimization work. 

Failure to solve the correctness issues first, he said, will just lead to more problems as more features are added. He drew a parallel with Btrfs which, he said, didn't solve the ""known hard problems"" early and, as a result, is stuck with ""entrenched deficiencies"" that are nearly impossible to fix. If those known hard problems are not solved first with DAX, it may well end up in the same situation. 

He would also like to see optimization work focused on the general case, instead of on providing opt-out mechanisms for a few programs. Fixing performance issues rather than bypassing them will provide benefits for everybody, a better outcome than just enabling a few applications to implement their own optimized solutions. If, instead, those applications opt out, they will not benefit from core-code improvements and, consequently, those improvements will be less likely to happen. 

Pushing back on and delaying work that kernel developers would like to see merged is never a pleasant experience. That work was done for a reason; rejecting it often means that at least some of that work was done in vain, and hard feelings can often result. But experience has shown that resisting work that seems premature or not consistent with long-term goals leads to a better, more maintainable kernel in the long run. The DAX infrastructure is going to have to serve as an important kernel-supported approach to persistent memory for a long time; the community cannot afford to get this one wrong. So there may well be a solid case to be made for conservatism in this area for now.
