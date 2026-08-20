---
title: "Bye-bye bdflush()"
url: https://lwn.net/Articles/861431/
date: "July 5, 2021"
category: "System calls-bdflush"
author: "By Jonathan Corbet July 5, 2021"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
July 5, 2021

The addition of system calls to the Linux kernel is a routine affair; it happens during almost every merge window. The removal of system calls, instead, is much more uncommon. That appears likely to happen soon, though, as discussions proceed on the removal of [`bdflush()`](<https://man7.org/linux/man-pages/man2/bdflush.2.html>). Read on for a look at the purpose and history of this obscure system call and to learn whether you will miss it (you won't). 

Linux, like most operating systems, buffers filesystem I/O through memory; a [`write()`](<https://man7.org/linux/man-pages/man2/write.2.html>) call results in a memory copy into the kernel's page cache, but does not immediately result in a write to the underlying block storage device. This buffering is necessary for writes of anything other than complete blocks; it is also important for filesystem performance. Deferring block writes can allow operations to be coalesced, provide opportunities for better on-disk file layout, and enables the batching of operations. 

Buffered file data cannot be allowed to live in memory forever, though; eventually the system must arrange for it to be flushed back to disk. Even the 0.01 Linux release included a version of the [`sync()`](<https://man7.org/linux/man-pages/man2/sync.2.html>) system call, which forces all cached filesystem data to be written out. While the kernel would flush some buffers when the buffer cache (which preceded the page cache and was a fixed-size array at that time) filled up, there was no provision for regularly ensuring that all buffers were pushed out to disk. That task was, if your editor's memory serves, handled by a user-space process that would occasionally wake up and call `sync()`. 

There are advantages to handling this task in the kernel, though; it has a much better idea of the state of both the buffer cache and the underlying devices. As a step in that direction, the `bdflush()` system call was added to the [0.99.14y release](<https://git.kernel.org/pub/scm/linux/kernel/git/history/history.git/commit/?h=0.99.14y&id=f614125ef74543bd223366c0c0d2a29be658b78e>) on February 2, 1994. (This was [a different era](<http://web.soccerlab.polymtl.ca/ser-repos/public/data-sets/linux-kernels/kernel-history/1994.html>) of kernel development; the preceding 0.99.14x release came out seven hours earlier, and 0.99.14z came out nine hours later). That implementation was not particularly useful, though; all it did was return a "not implemented" error. An actual `bdflush()` implementation was not added until the [1.1.3 development kernel](<https://git.kernel.org/pub/scm/linux/kernel/git/history/history.git/commit/?h=1.1.3&id=7450aa7e680c0037033c4f4e5746c66738c727ca>) in April 1994. 

It must be said that `bdflush()` was a strange system call. It was defined as: 

```
int bdflush(int func, long data);
```

If `func` was zero, `bdflush()` would never return; instead, it would loop within the kernel, occasionally flushing out dirty buffers. In essence, a user-space process would become the kernel buffer-flushing thread by making that call; these were the days before proper kernel threads, after all. Passing `func` as one would cause some buffers to be flushed immediately. Higher values of `func` would either read or write the value of a control parameter for the flushing thread; these included the percentage of dirty buffers needed to activate flushing, the number of blocks to write in each cycle, etc. 

While `bdflush()` was an improvement, there were a number of problems with it as well. One of those was that it relied on user space for a critical kernel function; if no process ever set itself up with `bdflush()`, or if that process were killed, bad things would happen. In the [1.3.50 development release](<https://git.kernel.org/pub/scm/linux/kernel/git/history/history.git/commit/?h=1.3.50&id=22accfc2b4fe4bc6a635c70c1a05a9a80abc81ea>) (December 1995), the kernel was changed to automatically create a kernel thread (something it could do at that point) to do the flushing work. User space could still call `bdflush()` to tweak the various parameters, but an attempt to run as the flushing daemon would turn into an immediate call to `exit()`; that caused the update process started by older init systems to "work", avoiding boot-time unhappiness. 

Another problem with `bdflush()` — or, more specifically, with the underlying implementation — since the beginning is that it was a single thread. As Linux grew in popularity and found itself on bigger systems, that single thread became an increasingly severe bottleneck. If you have a number of drives on a system, it will eventually take multiple threads to keep them all busy. Thus Andrew Morton replaced the remaining `bdflush()` infrastructure entirely in 2002 for the [2.5.8 development kernel](<https://mirrors.edge.kernel.org/pub/linux/kernel/v2.5/ChangeLog-2.5.8>); in its place was a new set of kernel threads called `pdflush`. Each `pdflush` thread was dedicated to a separate physical drive, providing a much-needed scalability improvement. 

In December 2002, Morton merged a patch from Robert Love formally deprecating the `bdflush()` system call, promising that it ""will be removed in a future kernel"". The `pdflush` threads were [removed in 2009](<https://git.kernel.org/linus/d0bceac747b5>) (for [2.6.32](<https://lwn.net/Articles/364928/>)) in favor of a rather more elaborate, workqueue-based, writeback-control mechanism; those can still be seen in the form of kernel threads with names like `kworker/u8:3-flush-259:0`. Meanwhile, though, `bdflush()` lives on in current kernels, even though it has not done anything for many years. 

Now, however, Eric Biederman is [proposing](<https://lwn.net/ml/linux-kernel/87sg10quue.fsf_-_@disp2133/>) to remove `bdflush()` entirely as part of a larger project he has to rework the kernel's `exit()` code. Given that this system call does nothing, was never widely used in the first place, and has been deprecated for nearly 19 years, one might confidently conclude that there are no users left. As it turns out, though, Geert Uytterhoeven [has an old m68k image](<https://lwn.net/ml/linux-arch/CAMuHMdXSU6_98NbC1UWTT_kmwxD=6Ha5LJxFAtbSuD=y78nASg@mail.gmail.com/>) that he occasionally boots, presumably on days when he is overcome with nostalgia. Michael Schmitz [demonstrated](<https://lwn.net/ml/linux-arch/36123b5d-daa0-6c2b-f2d4-a942f069fd54@gmail.com/>), though, that said image still boots successfully in the absence of `bdflush()`, so it is not an impediment to the system call's removal. 

There are no other known users of `bdflush()` out there, so there would appear to be nothing preventing this removal from happening. At that point, it will be the first system call removed since late 2019, when `sysctl()` was deleted — by the same Eric Biederman. It would be surprising to see that happen in 5.14, though, given how recently this patch was posted. This system call has endured for almost 19 years after it ceased to be useful; keeping it for another two months until 5.15 does not seem like much of an imposition.
