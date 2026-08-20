---
title: Files with negative offsets
url: https://lwn.net/Articles/138063/
date: "May 31, 2005"
category: Filesystems; x86 64
author: 
---

Every open file on a Linux system has an associated offset - the current read or write position within that file. The virtual filesystem code, when dealing with file positions, performs some basic checks, such as ensuring that the position is not negative. After all, what sense does it make to talk about a file position before the beginning of the file? 

As it turns out, [there is a situation](<https://lwn.net/Articles/138064/>) where a negative file position makes sense. Special files (such as `/dev/mem` and `/dev/kmem`) provide a window into the system's main memory. The "position" within these files corresponds to the address of the memory of interest. The interesting thing is that, on the x86_64 platform, addresses can be negative numbers. 

This comes about as follows: this architecture currently uses a 48-bit address space. The hardware sign-extends the uppermost bit, however, so any address with that bit set will turn into a negative number. The x86_64 Linux port uses the upper bit to mark kernel space, so kernel addresses are, in fact, negative. A quick look at `/proc/kallsyms` confirms this: 

```
ffffffff80100000 T startup_32
        ffffffff80100100 T startup_64
        ffffffff801001a0 T initial_code
        ffffffff801001a8 T init_rsp
        ffffffff801001b0 T early_idt_handler
        ...
```

The end result is that using `/dev/kmem` on an x86_64 system is difficult; any attempt to seek into kernel space will yield an error. 

The clear fix is to modify the VFS layer to let negative file positions be passed through to the underlying filesystem or device driver. The problem with doing that in a general way, however, is that not all code (especially in drivers) is prepared to deal with a negative offset. Suddenly exposing that code to negative offsets could open up no end of bugs and security problems. So the real solution, as [worked out by Al Viro and Linus Torvalds](<https://lwn.net/Articles/138066/>), is to add a new flag for the `file` structure called `FMODE_ANY_OFFSET`. This flag can only be set within the kernel; user space has no access to it. So the `/dev/kmem` driver will be able to set the flag and work with the full range of offsets, but, for the rest of the system, nothing will change.
