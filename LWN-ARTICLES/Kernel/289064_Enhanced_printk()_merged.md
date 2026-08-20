---
title: Enhanced printk() merged
url: https://lwn.net/Articles/289064/
date: "July 9, 2008"
category: printk
author: "By Jake Edge July 9, 2008"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
July 9, 2008

A change very late in the development cycle for 2.6.26 provides a framework for extending `printk()` to handle new kinds of arguments. Linus Torvalds just merged the change--_after_ -rc9--presumably partially because he knew he could trust the author, but also because it should have no effect on the kernel. It will provide for better debugging output once code is changed to take advantage of it. 

The core idea is to extend `printk()` so that kernel data structures can be formatted in kernel-specific ways. In order to get some compile-time checking, the `%p` format specifier has been overloaded. For example, `%pI` might be used to indicate that the associated pointer is to be formatted as a `struct inode`, which could print the most interesting fields of that structure. GCC will be able to check for the presence of a pointer argument, but because it does not understand the `I` part, cannot enforce that it is a pointer of the right type. 

Extending `printk()` in this manner allowed Torvalds--who authored the [patch](<http://ozlabs.org/pipermail/linuxppc-dev/2008-July/059271.html>)--to add two new types to `printk()`: `%pS` for symbolic pointers and `%pF` for symbolic function pointers. In both cases, the code uses `kallsyms` to turn the pointer value into a symbol name. Instead of a kernel developer having to read long address strings and then trying to find them in the system map, the kernel will do that work for them. 

The `%pF` specifier is for architectures like ppc and ia64 that use function descriptors rather than pointers. For those architectures, a function pointer points to a structure that contains the actual function address. By using the `%pF` specifier, the proper dereferencing is done. 

As an example of how the augmented `printk()` could be used, Torvalds [converted `printk_address()`](<http://marc.info/?l=linux-ia64&m=121529722828758&w=2>). The `CONFIG_KALLSYMS` dependency and the `kallsyms_lookup()` were removed, essentially leaving a one-line function: 

```
printk(" [<%016lx>] %s%pS\n", address, reliable ? "": "? ", (void *) address);
```

If `kallsyms` is not present, the new `printk()` just reverts to printing the address in hexadecimal, which allows the special case handling to be done there. 

The clear intent is to allow additional extensions to `printk()` to support other kernel data structures. The change to `vsprintf()`, which underlies `printk()`, actually allows for any sequence of alphanumeric characters to appear after the `%p`. The new `pointer()` helper function currently only implements the two new specifiers, but others have been mentioned. 

The mostly likely additions are for things like IPv4, IPv6, and MAC addresses. Torvalds specifically [mentions](<https://lwn.net/Articles/289191/>) using `%p6N` as a possibility for IPv6 addresses. Some would rather have seen a different syntax be used, `%p{feature}` was [suggested](<https://lwn.net/Articles/289196/>), but that would conflict with some current uses of `%p` in the kernel. Torvalds is [happy with his choice](<https://lwn.net/Articles/289197/>): 

I _expressly_ chose '%p[alphanumeric]*' because it's basically totally insane to have that in a *real* printk() string: the end result would be totally unreadable. 

The patch took an interesting route to the kernel, with much of the discussion evidently going on in private between Torvalds, Andrew Morton, and others before popping up on the linuxppc-dev and linux-ia64 mailing lists. The patch itself has not been posted to linux-kernel in its complete form, but was committed on July 6. While it is a bit strange to see such a change this late in the development cycle, it is a change that should have no impact as there are [no plans](<http://ozlabs.org/pipermail/linuxppc-dev/2008-July/059437.html>) to actually _use_ the new specifiers in 2.6.26.
