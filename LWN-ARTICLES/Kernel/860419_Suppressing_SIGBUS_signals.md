---
title: Suppressing SIGBUS signals
url: https://lwn.net/Articles/860419/
date: "June 25, 2021"
category: "System calls-mmap"
author: "By Jonathan Corbet June 25, 2021"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
June 25, 2021

The [`mmap()` system call](<https://man7.org/linux/man-pages/man2/mmap.2.html>) creates a mapping for a range of virtual addresses; it has a long list of options controlling just how that mapping should work. Ming Lin is [proposing](<https://lwn.net/ml/linux-kernel/1622792602-40459-1-git-send-email-mlin@kernel.org/>) the addition of yet another option, called `MAP_NOSIGBUS`, which changes the kernel's response when a process accesses an unmapped address. What this option does is relatively easy to understand; why it is useful takes a bit more explanation. 

Normally, when a process performs an operation involving memory, it expects the desired data to be read from or written to the requested location. Sometimes, though, things can go wrong, resulting in the delivery of a fatal (by default) signal to the process. A "segmentation violation" (`SIGSEGV`) signal is generated in response to an attempt to access a valid memory address in a way that is contrary to its protection — writing to read-only memory, for example. Attempting to access an address that is invalid, instead, results in a "bus error" (`SIGBUS`). Bus errors can be provoked in a number of ways, including using an improperly aligned address or an address that is not mapped at all. If a process uses `mmap()` to create a mapping that extends beyond the end of the backing file, attempts to access the pages past the end of the file will result in `SIGBUS` signals. 

If, however, a memory range has been mapped with the proposed `MAP_NOSIGBUS` flag, `SIGBUS` signals will no longer be generated in response to an invalid address that lies within the mapped area. Instead, the guilty process will get a new page filled with zeroes. If the mapped area is backed up by a file on disk, the new page will not be added to that file. To a first approximation, the new option simply makes `SIGBUS` signals go away, with the process never even knowing that it had tried to access an invalid address. 

#### OK...but why?

This behavior may seem like a strange thing to want. One would not normally expect a mapped area to contain invalid addresses within it, and one ordinarily wants to know if a program is generating and using invalid addresses. As it happens, mapped areas _can_ contain invalid addresses in one normal use case: if that area is mapping a file, and it extends beyond the end of the file on disk. Attempts to access pages beyond the end of the file will generate a `SIGBUS` signal; this situation can be avoided by extending the file before attempting to access it through the mapping. 

`MAP_NOSIGBUS` is explicitly incompatible with that way of working, though; since the zero-filled pages that it creates in response to invalid addresses are not connected to the backing file, it makes extending the file without redoing the mapping impossible. Instead, this option exists to address another problem: graphical clients that can, accidentally or intentionally, cause a compositor to crash. 

Graphical applications often have to communicate large amounts of data to the compositor. An efficient way of doing this can be to map a file and pass a descriptor to the compositor; that file (which can live in a memory-only filesystem) becomes a shared-memory segment between the two processes. If, however, the client process then calls [`ftruncate()`](<https://man7.org/linux/man-pages/man2/ftruncate.2.html>) to shorten the file, the result is a mapping (in the compositor) that extends beyond the end of that file. If the compositor tries to access the shared-memory segment beyond the new end of the file, it will get a `SIGBUS` signal; in the absence of measures taken to the contrary, that will cause the compositor to crash, which is the sort of thing that user-experience developers usually make at least a modest effort to avoid. The `SIGBUS` signal can be caught and handled in the compositor, but that can be complex and hard to get right. 

As Simon Ser, who works on Wayland compositors, [noted](<https://lwn.net/ml/linux-mm/vs1Us2sm4qmfvLOqNat0-r16GyfmWzqUzQ4KHbXJwEcjhzeoQ4sBTxx7QXDG9B6zk5AeT7FsNb3CSr94LaKy6Novh1fbbw8D_BBxYsbPLms=@emersion.fr/>) back in April, there is another mechanism for passing data between the two processes: the [memfd](<https://lwn.net/Articles/593918/>) abstraction. A memfd can be "sealed", meaning that the creator cannot shrink it as described above (or, indeed, change it at all); the recipient, knowing that the segment will not change unexpectedly, can access it safely. But, as Ser points out, no compositor requires the use of sealed memfds because there are clients that are unwilling or unable to use them. So compositors must either jump through the `SIGBUS`-handling hoops or risk filling the disk with embarrassing core dumps. 

But if the compositor could map a segment in a way that wouldn't create `SIGBUS` signals on invalid addresses, this whole problem would go away. Ser suggested looking at the `__MAP_NOFAULT` flag supported by OpenBSD as a possible solution. At the beginning of June, Lin responded with [an implementation](<https://lwn.net/ml/linux-kernel/1622589753-9206-1-git-send-email-mlin@kernel.org/>) of `MAP_NOSIGBUS`, which differs from `__MAP_NOFAULT` in a number of ways. The initial implementation only worked for the in-memory tmpfs filesystem, but Hugh Dickins [objected](<https://lwn.net/ml/linux-kernel/alpine.LSU.2.11.2106011913590.3353@eggly.anvils/>), saying that it should apply to any mapping; the second (and current revision) reflects that criticism and works regardless of the backing store behind a mapping. 

#### Limitations

One significant limitation of the current implementation is that it only works for `MAP_PRIVATE` mappings — that seems like it could be a fatal flaw for a mechanism that is meant for use with mappings shared between clients and a compositor. But, as Ser [explained](<https://lwn.net/ml/linux-mm/lpi4uT69AFMwtmWtwW_qJAmYm_r0jRikL11G_zI4X7wq--6Jtpiej8kGn8gePfv0Dtn4VmzsOqT2Q5-L3ca2niDi0nlC0nVYphbFBnNJnw0=@emersion.fr/>), private mappings will work in almost all cases; since the data transfer is one-way from the client to the compositor, the mapping can be read-only on the compositor side. The big exception is screen capture, which will still have to be handled specially as long as shared mappings are not supported. So the solution is not complete, but 90% is a big step in the right direction. 

The second version of the patch set has seen relatively little discussion; it seems that the developers who care about it are relatively happy with its current condition (though Kirill Shutemov was [heard to grumble a bit](<https://lwn.net/ml/linux-kernel/20210604152407.ouchyfuxjvchfroe@box/>) about ""one-user features""). There are never any guarantees, but there does seem to be a reasonable chance that this change could be merged as early as the 5.14 release.
