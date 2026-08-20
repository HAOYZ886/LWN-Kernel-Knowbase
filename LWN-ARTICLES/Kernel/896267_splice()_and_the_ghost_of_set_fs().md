---
title: splice() and the ghost of set_fs()
url: https://lwn.net/Articles/896267/
date: "May 26, 2022"
category: "Development model-Regressions; set fs"
author: "By Jonathan Corbet May 26, 2022"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
May 26, 2022

The normal rule of kernel development is that the creation of user-space regressions is not allowed; a patch that breaks a previously working application must be either fixed or reverted. There are exceptions, though, including [a 5.10 patch](<https://git.kernel.org/linus/36e2c7421f02>) that has been turning up regressions ever since. The story that emerges here shows what can happen when the goals of stability, avoiding security problems, and code cleanup run into conflict. 

The `set_fs()` function was added to the kernel early in its history; it was not in the initial 0.01 release, but was added before the 0.10 release in late 1991. Normally, kernel code that is intended to access user-space memory will generate an error if it attempts to access kernel space instead; this restriction prevents, for example, attempts by an attacker to access kernel memory via system calls. A call to `set_fs(KERNEL_DS)` can be used to lift the restriction when the need arises; a common use case for `set_fs()` is to be able to perform file I/O from within the kernel. Calling `set_fs(USER_DS)` puts the restriction back. 

The problem with `set_fs()` is that it turns out to be easy to forget the second `set_fs()` call to restore the protection of kernel space, leading directly to the "total compromise" scenario that kernel developers will normally take some pains to avoid. Numerous such bugs have been fixed over the years, but it had long been clear that the real solution was to just get rid of `set_fs()` entirely and adopt safer ways of accessing kernel memory when needed. 

Developers (and Christoph Hellwig in particular) got more serious about this objective in 2020 and [made a determined push](<https://lwn.net/Articles/832121/>) to eliminate `set_fs()` entirely. Much of this work went into 5.10, though the final bits of the `set_fs()` infrastructure were only removed in 5.18. Back in 2020, though, one question that provoked some discussion was what should be done about [`splice()`](<https://man7.org/linux/man-pages/man2/splice.2.html>). 

The `splice()` system call will connect an open file descriptor to a pipe, then move data between the two for as long as the data stream lasts. This movement happens entirely within the kernel, potentially eliminating the need for large numbers of system calls; in some settings, it can provide a significant performance improvement. By its nature, `splice()` often has to move data to and from buffers that are in kernel space; to make that possible, it used `set_fs()`. 

Hellwig duly came up with a new implementation that would keep `splice()` working in the absence of `set_fs()`, but Linus Torvalds [rejected it](<https://lwn.net/ml/linux-kernel/CAHk-=wit9enePELG=-HnLsr0nY5bucFNjqAqWoFTuYDGR1P4KA@mail.gmail.com/>), saying that he didn't like the ""complexity and messiness"" of the implementation. But he also made it clear that he didn't feel the need to guarantee that `splice()` would keep working at all; he felt that making `splice()` work by default on most file types led to a number of security issues. Later in 2020, for example, he [said](<https://lwn.net/ml/linux-kernel/CAHk-%3DwiXQVE_jGN0ajk%2BKm925WSbCL16mAZ-UXNkp%2Bnkc1nuQw%40mail.gmail.com/>): 

> I'd rather limit splice (and kernel_read too, for that matter) as much as possible. It was a mistake originally to allow it everywhere, and it's come back to bite us. 
> 
> So I'd rather have people notice these odd corner cases and get them fixed one by one than just say "anything goes". 

So the patches that went into 5.10 ended up breaking `splice()` for any file type that did not have explicit support for the new way of doing things; the idea was that the important cases would be noticed and fixed over time. That has indeed happened; if one looks for patches committed as explicit fixes to the disabling of `splice()` support, one finds fixes for [the AFS filesystem](<https://git.kernel.org/linus/06a17bbe1d47>), [the 9p filesystem](<https://git.kernel.org/linus/cf03f316ad20>), [the orangefs filesystem](<https://git.kernel.org/linus/c1048828c3db>), [`/proc/mountinfo`](<https://git.kernel.org/linus/14e3e989f6a5>), [the TTY subsystem](<https://git.kernel.org/linus/dd78b0c483e3>), [kernfs](<https://git.kernel.org/linus/f2d6c2708bd8>), [`sendfile()`](<https://git.kernel.org/linus/f8ad8187c3b5>), [the nilfs2 filesystem](<https://git.kernel.org/linus/a35d8f016e0b>), and [the JFFS2 filesystem](<https://git.kernel.org/linus/42984af09afc>). 

Most recently, Jens Axboe [reported](<https://lwn.net/ml/linux-kernel/20220519193133.194138-1-axboe@kernel.dk/>) that `splice()` no longer worked on `/dev/random` or `/dev/urandom`; he included a patch to fix the problem as well. These patches were later [reworked](<https://lwn.net/ml/linux-kernel/20220520094459.116240-1-Jason@zx2c4.com/>) by random-number-generator maintainer Jason Donenfeld and were applied to the mainline during the 5.19 merge window. Along the way, Donenfeld [observed](<https://lwn.net/ml/linux-kernel/Yoey+FOYO69lS5qP@zx2c4.com/>) that the necessary changes resulted in a performance regression of about 3% when reading from `/dev/urandom`. That led him to ask whether the fix was something that was needed at all; after some discussion, Axboe [gave him the lecture](<https://lwn.net/ml/linux-kernel/72344aad-b5ad-b317-d36d-385cb16d5204@kernel.dk/>) on regressions: 

> If you have an application that is written using eg splice from /dev/urandom, then it should indeed be safe to expect that it will indeed continue working. If we have one core tenet in the kernel it's that you should ALWAYS be able to upgrade your kernel and not have any breakage in terms of userspace ABI. Obviously that can happen sometimes, but I think this one is exactly the poster child of breakage that should NOT happen. We took away a feature that someone depended on. 

That is the sort of breakage that did indeed happen but, in this case, a change was made knowing that this kind of problem would result. Hellwig [said](<https://lwn.net/ml/linux-kernel/20220520060244.GA16503@lst.de/>) in response to Axboe's patch set that ""compared to my initial fears the fallout actually isn't that bad"", but a perusal of the above list of fixes might lead one to a different conclusion. 

The removal of `set_fs()` is, in many ways, a model for what the kernel development process can do. A fundamental piece of low-level structure that had been deeply wired into the kernel since the beginning was replaced with a much safer alternative without breaking the project's pace of a stable release every nine or ten weeks. The steady stream of regressions resulting from this change, though, is not what the project sets out to do — and it seems certain that this particular gift has not yet stopped giving. 

The decision to take this path was driven by a fear of security problems, based on the past history of the `splice()` system call. If those fears are still justified (and they might well be; consider, for example, that `splice()` was a part of the ["Dirty Pipe" vulnerability](<https://dirtypipe.cm4all.com/>) reported earlier this year), then refusing to make all existing `splice()` implementations just work without `set_fs()` may have prevented far worse regressions than the ones we have seen. Having to fix a filesystem is annoying; having to endure yet another security drill for a branded vulnerability with a silly name is rather more so. 

There is no way of knowing whether that is how things would have gone in this case. But it is true that this type of episode makes the kernel's "no regressions" rule look a bit more like just a guideline. It does not take too many of those to create breakage to the project's reputation that is hard to splice back together.
