---
title: mseal() gets closer
url: https://lwn.net/Articles/958438/
date: "January 19, 2024"
category: "Releases-6.10; Security-System calls"
author: "By Jonathan Corbet January 19, 2024"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
January 19, 2024

The [proposed `mseal()` system call](<https://lwn.net/Articles/948129/>) stirred up some controversy when it was first posted in October 2023. Since then, it has been evolving in a quieter fashion, and seems to have reached a point where the relevant commenters are willing to accept it. Should `mseal()` be merged in a future development cycle, it will look rather different than it did at the outset. 

As a reminder, `mseal()` was created as a way of preventing changes to portions of the virtual address space. It is meant to thwart attacks that depend on changing memory that is supposed to be read-only or otherwise messing with a process's idea of how its memory is laid out. An attacker who can change memory permissions or mappings may, for example, be able to circumvent control-flow-integrity protections. By using `mseal()`, a process can prevent changes of that type from being made. The initial user is expected to be the Chrome browser, where it will be used to further harden the program against memory-based attacks. 

`mseal()`, as proposed in October, had this prototype: 

```
int mseal(void *addr, size_t len, unsigned int types, unsigned int flags);
```

The `types` parameter, which allowed the caller to fine-tune the changes that `mseal()` would prohibit, was one of the more controversial features, with a number of people questioning why anything other than an outright ban on changes would be useful. Even so, [version 2](<https://lwn.net/ml/linux-mm/20231017090815.1067790-1-jeffxu@chromium.org/>) (posted shortly after the first version) and [version 3](<https://lwn.net/ml/linux-kernel/20231212231706.2680890-1-jeffxu@chromium.org/>) (posted in mid-December) retained that parameter. In response to the latter posting, Linus Torvalds [reiterated](<https://lwn.net/ml/linux-kernel/CAHk-=wgn02cpoFEDQGgS+5BUqA2z-=Ks9+PNd-pEJy8h+NOs5g@mail.gmail.com/>) his dislike of that aspect of the API, and asked: ""I want to know why we don't just do the BSD immutable thing, and why we need this multi-level sealing thing"". 

Chrome developer Stephen Röttger [answered](<https://lwn.net/ml/linux-kernel/CAEAAPHZpYXHNPdca+xfj77bwYaL6PY-c_oQ54r+=wtJa6_hmCA@mail.gmail.com/>) that Chrome needed the ability to allow `[madvise](<https://man7.org/linux/man-pages/man2/madvise.2.html>)(MADV_DONTNEED)` in specific places where the region could otherwise be sealed. This operation was forbidden in sealed memory because it is essentially a mapping change; it discards the underlying memory, which (for anonymous pages) will be refilled with zeroes if it is accessed again. It is useful for (for example) discarding unneeded cached data, but it also has the potential to create surprises. In the Chrome case, the `type` argument was used to allow `MADV_DONTNEED` on writable anonymous memory — memory that the process has the ability to write directly even when it is sealed. Torvalds [replied](<https://lwn.net/ml/linux-kernel/CAHk-=wiVhHmnXviy1xqStLRozC4ziSugTk=1JOc8ORWd2_0h7g@mail.gmail.com/>) that the proper solution was for `mseal()` to only allow `MADV_DONTNEED` if the mapping in question is writable. Indeed, he thought that restriction might make sense even in the absence of sealing. 

As a result of that discussion, [version 4](<https://lwn.net/ml/linux-kernel/20240104185138.169307-1-jeffxu@chromium.org/>), posted in early January, implemented the new semantics with regard to `MADV_DONTNEED`. This version also finally dropped the `types` parameter; memory is now either sealed or not. Torvalds [was satisfied](<https://lwn.net/ml/linux-kernel/CAHk-=wiy0nHG9+3rXzQa=W8gM8F6-MhsHrs_ZqWaHtjmPK4=FA@mail.gmail.com/>) by the changes; he declared that ""this seems all reasonable to me now"" and withdrew from the discussion. The [fifth version](<https://lwn.net/ml/linux-mm/20240109154547.1839886-1-jeffxu@chromium.org/>) brought only small changes, suggesting that the major concerns have been addressed; Kees Cook [noted](<https://lwn.net/ml/linux-mm/202401091144.2C203AA7@keescook/>) that ""this code is looking to land"". Since then, [version 6](<https://lwn.net/ml/linux-mm/20240111234142.2944934-1-jeffxu@chromium.org/>) was posted with a few more small changes. 

If `mseal()` is merged in this form, its prototype will be: 

```
int mseal(void *addr, size_t len, unsigned long flags);
```

The `addr` and `len` describe the range of memory to be sealed; the `flags` argument is currently unused and must be zero. It will only be available on 64-bit systems. [This documentation patch](<https://lwn.net/ml/linux-mm/20240111234142.2944934-5-jeffxu@chromium.org/>) contains more information about its use. 

So this story may have run its course, but there is still one aspect of it that has been somewhat swept under the rug. OpenBSD has a similar system call, [`mimmutable()`](<https://lwn.net/Articles/915640/>), that has been in place since 2022. It, too, prevents modifications to a specific range of the address space. Over the course of the conversation, simply implementing `mimmutable()` for Linux has been suggested a number of times. Jeff Xu, the developer of `mseal()`, has always shrugged off that suggestion, to the point that Theo de Raadt, the creator of `mimmutable()`, [suggested](<https://lwn.net/ml/linux-kernel/56221.1702566263@cvs.openbsd.org/>) that ""maybe this proposal should be using the name chromesyscall() instead"". It seems that implementing `mimmutable()` for Linux has never been seriously considered. 

As `mseal()` has gotten simpler, though, the features that differentiated it from `mimmutable()` have melted away, to the point that they do almost the same thing. About the only difference is that `mimmutable()` allows downgrading permissions (setting memory read-only even if it has been sealed), while `mseal()` does not; OpenBSD [may yet remove](<https://lwn.net/ml/linux-kernel/58421.1702516178@cvs.openbsd.org/>) that feature, though, further reducing the difference in semantics between the two system calls. Given that, it may be worth asking, one more time, why Linux doesn't just adopt the existing interface and add `mimmutable()`. It is not a question that has been directly addressed. 

Possible answers do exist. `mseal()` carries the `flags` parameter that [long experience](<https://lwn.net/Articles/585415/>) says is a good idea, even if the immediate need for it is not apparent. It may also be that the use of this system call will always be so specialized and low-level that any code using it will need to be system-specific in any case, in which case there may be little value in using the same name. Finally, adding an `mimmutable()` wrapper around `mseal()` in the C library would be an almost trivial undertaking if it were deemed worthwhile. 

If and when `mseal()` is merged, it will initially only benefit the Chrome browser (and its small band of users). As the `mseal()` cover letter points out, though, Röttger is working on adding support to the GNU C Library so that most programs would be able to run with a fair amount of sealing automatically applied. That would greatly increase the use of this new system call, and the ability to use it in the C library would increase confidence that the API is correct. That seems likely to truly seal the deal.
