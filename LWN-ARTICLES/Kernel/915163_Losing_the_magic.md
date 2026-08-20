---
title: Losing the magic
url: https://lwn.net/Articles/915163/
date: "December 5, 2022"
category: "Development model-Patterns"
author: "By Jonathan Corbet December 5, 2022"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
December 5, 2022

The kernel project is now more than three decades old; over that time, a number of development practices have come and gone. Once upon a time, the use of "magic numbers" to identify kernel data structures was seen as a good way to help detect and debug problems. Over the years, though, the use of magic numbers has gone into decline; [this patch set](<https://lwn.net/ml/linux-kernel/cover.1668128257.git.nabijaczleweli@nabijaczleweli.xyz/>) from Ahelenia Ziemiańska may be an indication that the reign of magic numbers may be reaching its end. 

A magic number is simply a specific constant value that is placed within a structure, typically as the first member, to identify the type of that structure. When structures are labeled in this way, in-kernel debugging code can check the magic number and raise the alarm if the expected value is not found, thus detecting problems related to type confusion or data corruption. These numbers can also be located in hexadecimal data dumps (stack contents, for example) to identify known data structures. 

The use of magic numbers in the kernel appears to have had its origin in the filesystem code, where it was initially used to identify (and verify) the superblock in the disk image. Even the 0.10 kernel release included [a test against `SUPER_MAGIC`](<https://elixir.bootlin.com/linux/0.10/source/fs/super.c#L131>) (`0x137f`) to verify that the boot disk was, indeed, a Minix filesystem. Other filesystems came later, starting with the "first extended" (ext), which used `0x137d` for its `EXT_SUPER_MAGIC` value in the 0.96c release in July 1992. 

In the 0.99 release (December 1992), the [`sk_buff` structure](<https://elixir.bootlin.com/linux/0.99/source/net/tcp/sock.h#L155>) that is still used in the networking subsystem to hold packets was rather smaller than [it is now](<https://elixir.bootlin.com/linux/v6.1-rc5/source/include/linux/skbuff.h#L728>), but it did gain a `magic` field to identify the queue a packet was expected to be in. Toward the middle of 1993, the 0.99.11 release acquired an updated [`kmalloc()` implementation](<https://elixir.bootlin.com/linux/0.99.11/source/lib/malloc.c>) that sprinkled magic numbers around as a debugging aid. That release, incidentally, is also the one where an attempt was made to use C++ to build the kernel; that only lasted until 0.99.13, a couple of months later. 

The use of magic numbers in the kernel grew slowly after that. The 1.1.13 release, in May 1994, added [a file called `MAGIC`](<https://elixir.bootlin.com/linux/1.1.13/source/MAGIC>) in the top-level directory to keep track of the various numbers in use; it listed eight such numbers. This file was, incidentally, nearly the first documentation file in the kernel beyond the basic installation information; the kernel would not gain a directory for documentation until 1.3.22 in 1995. In this new file, Ted Ts'o wrote: 

> It is a *very* good idea to protect kernel data structures with magic numbers. This allows you to check at run time whether (a) a structure has been clobbered, or (b) you've passed the wrong structure to a routine. This last is especially useful --- particlarly when you are passing pointers to structures via a void * pointer. The tty code, for example, does this frequently to pass driver-specific and line discipline-specific structures back and forth. 

The file went on to ask developers to follow this practice for future additions to the kernel. (For the curious: the typo of "particularly" lasted until 1.1.42 in August 1994; otherwise that text persists to this day). 

The 1.3.99 release saw the movement of `MAGIC` to [`Documentation/magic-number.txt`](<https://elixir.bootlin.com/linux/1.3.99/source/Documentation/magic-number.txt>), perhaps as part of a general cleaning-up prior to the imminent 2.0 release. Some developers, at least, had clearly taken Ts'o's advice; there were, at this point, 21 entries in that file. The [2.2.0 version](<https://elixir.bootlin.com/linux/2.2.0/source/Documentation/magic-number.txt>) (January 1999) held 51 entries. Magic numbers appeared to be an established kernel-development practice. 

The 2.4.0 release came almost exactly two years later. [The 2.4 version of `magic-number.txt`](<https://elixir.bootlin.com/linux/2.4.0/source/Documentation/magic-number.txt>) was, except for one small change, identical to the 2.2 version; no new magic numbers had been added. That doesn't necessarily reflect a change in development practices so much as the eternal habit of letting documentation go out of date. Some effort went into updating the file during the 2.5 development series, and the [2.6.0 version](<https://elixir.bootlin.com/linux/v2.6.0/source/Documentation/magic-number.txt>) contained an even 100 magic numbers. For the rest of the 2.6.x series, though, the only changes were small tweaks and the removal of a couple of obsolete entries; `Documentation/magic-number.txt` began to shrink. 

In fact, no additions to that file have been made in the entire Git history. In 2016 the file was converted to the RST format and moved into the nascent development-process manual. It mostly sat idle and unnoticed until earlier this year, when the file began to shrink; the [6.1 version of `Documentation/process/magic-number.rst`](<https://elixir.bootlin.com/linux/v6.1-rc5/source/Documentation/process/magic-number.rst>) is down to 14 entries. Where has the magic gone? 

This change is the result of Ziemiańska's work, which is aimed at removing this file entirely; the current patch set describes it as ""a low-value historical relic at best and misleading cruft at worst"". In that series, Ziemiańska systematically deletes the final entries, often removing the associated structure fields and magic-number checks in the code, until the file is empty; the [final patch](<https://lwn.net/ml/linux-kernel/93e66d80680c52e8d04763fb2d86f841c69e32c9.1668128257.git.nabijaczleweli@nabijaczleweli.xyz/>) in the series simply deletes it. Chances are that it will not be missed. 

There is no clear point where the development community made a collective decision to move away from the magic-number practice; it just sort of faded away. The are probably a few reasons behind this change. The kernel community has, for many years now, tried to use type-safe interfaces rather than passing `void` pointers around, making it less likely that the wrong structure type will be passed into a function. Developers spend less time staring at hex dumps of data, preferring more structured output, tracepoints, and interactive debuggers as ways of tracking down problems. Debugging features in the kernel's memory allocators mean that many sorts of memory-corruption issues will be caught directly. Magic numbers are just not as helpful as they once were. 

Magic numbers will still have their place; they still, for example, can help filesystem code be confident that it is dealing with the correct type of filesystem image. Even there, though, the use of checksums for on-disk data structures should provide better protection against many types of problems. But, for the most part, kernel development has lost some of the magic it once had; as often happens, it slipped away when nobody was paying attention.
