---
title: Saying goodbye to set_fs()
url: https://lwn.net/Articles/832121/
date: "September 24, 2020"
category: set fs
author: "By Jonathan Corbet September 24, 2020"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
September 24, 2020

The `set_fs()` function dates back to the earliest days of the Linux kernel; it is a key part of the machinery that keeps user-space and kernel-space memory separated from each other. It is also easy to misuse and has been the source of various security problems over the years; kernel developers have long wanted to be rid of it. They won't completely get their wish in the 5.10 kernel but, as the result of work that has been quietly progressing for several months, the end of `set_fs()` will be easily visible at that point. 

[This 2017 article](<https://lwn.net/Articles/722267/>) describes `set_fs()` and its history in some detail. The short version is that `set_fs()` sets the location of the boundary between the user-space portion of the address space and the kernel's part. Any virtual address that is below the boundary set by the last `set_fs()` call on behalf of a given process is fair game for that process to access, though the memory permissions stored in the page tables still apply. Anything above that limit belongs to the kernel and is out of bounds. 

Normally, that boundary should be firmly fixed in place. When the need to move it arises, the reason is usually the same: some kernel subsystem needs to invoke a function that is intended to access user-space data, but on a kernel-space address. Think, for example, of the simple task of reading the contents of a file into a memory buffer; the `read()` system call will do that, but it also performs all of the usual access checks, meaning that it will refuse to read into a kernel-space buffer. If a kernel subsystem must perform such a read, it first calls `set_fs()` to disable those checks; if all goes well, it remembers to restore the old boundary with another `set_fs()` call when the work is done. 

Naturally, history has proved that all does not always go well. It's thus not surprising that the development community has wanted to rid itself of `set_fs()` for many years. It's also unsurprising that this hasn't happened, though. The kernel project does not lack for developers, but there is always a shortage of people who are willing and able to do this sort of deep infrastructural work; it tends to not feature highly in any company's marketing plan. So the task of removing `set_fs()` has languished for years. 

Recently, though, Christoph Hellwig has stepped up to this task and the kernel-wide cleaning-up that is required to get it done. 

For example, one might be surprised to find `set_fs()` calls in the core networking code, and even more surprised to learn that they were added in 2019, during the 5.3 development cycle. [The patch in question](<https://git.kernel.org/linus/0d01da6afc54>) added the ability for BPF programs to invoke the [`setsockopt()` and `getsockopt()` system calls](<https://man7.org/linux/man-pages/man2/setsockopt.2.html>). Those calls are normally invoked from user space, so they apply the usual access checks on any parameters passed to them; calls originating from BPF programs, though, will supply buffers in kernel space. Putting in a call to `set_fs()` in that case allowed those calls to work without further modification. 

Hellwig's plan for taking that `set_fs()` call back out involved the creation of a new `sockptr_t` type that can hold an address pointing into either kernel or user space: 

```
typedef struct {
    	union {
    	    void	*kernel;
    	    void __user	*user;
    	};
    	bool		is_kernel : 1;
        } sockptr_t;
```

Code that initializes a `sockptr_t` variable must specify whether the address is meant to refer to kernel or user space; a set of helper functions can then be used to copy data to and from that address without needing to worry further about where the destination buffer is — or to call `set_fs()`. As it turns out, `setsockopt()` and `getsockopt()` offer a lot of different options, so [a long patch series](<https://lwn.net/ml/linux-kernel/20200723060908.50081-1-hch@lst.de/>) was required to convert the relevant functions to `sockptr_t` addresses. At the end of the series, the `set_fs()` calls were removed. This series entered the mainline during the 5.9 merge window. 

Something that was _not_ merged was [an earlier version of this idea](<https://lwn.net/ml/linux-kernel/20200624162901.1814136-1-hch@lst.de/>), which was meant to be used throughout the kernel. Hellwig proposed the creation of a "universal pointer" type (`uptr`) that functioned like `sockptr_t`; it was accompanied by a pair of new `file_operations` methods that would work with those pointers. Then, any kernel subsystem that might need to perform I/O on both kernel-space and user-space pointers could be converted to use these new methods rather than calling `set_fs()`. 

Linus Torvalds [vetoed that idea](<https://lwn.net/ml/linux-kernel/CAHk-=wit9enePELG=-HnLsr0nY5bucFNjqAqWoFTuYDGR1P4KA@mail.gmail.com/>); he objected to the addition of the new type and `file_operations` methods, which he saw as temporary and unnecessary workarounds for the real problem. If somebody was going to bother to convert ordinary `read()` and `write()` calls to the new `read_uptr()` and `write_uptr()`, he asked, why wouldn't they just convert to the existing `read_iter()` and `write_iter()` methods instead? Those methods already handle the different address spaces just fine (through yet another union in [`struct iov_iter`](<https://elixir.bootlin.com/linux/v5.8.10/source/include/linux/uio.h#L30>) that tracks which type of address is in use); indeed, much of the work to remove `set_fs()` calls in various parts of the kernel has involved switching to `iov_iter`. So the `uptr` type fell by the wayside, but the `sockptr_t` was able to overcome Torvalds's opposition and was merged. 

Then, there is the `set_fs()` call that isn't actually there. In current kernels, the boundary between kernel and user space is established fairly late in the boot process (but before the `init` process is started). Before that happens, kernel functions that operate on user-space pointers will happily use kernel-space pointers instead; parts of the initialization code (dealing with the initial ramdisk, for example) depend on this behavior. Eliminating that implicit `set_fs()` call required [another patch series](<https://lwn.net/ml/linux-kernel/20200728163416.556521-1-hch@lst.de/>) creating a set of special helpers that is discarded once the bootstrap process is complete. This series, too, was merged for the 5.9 release. 

The final step, for the x86 and PowerPC architectures at least, is [this patch series](<https://lwn.net/ml/linux-kernel/20200903142242.925828-1-hch@lst.de/>) removing `set_fs()` entirely. Getting there requires tidying up a number of loose ends. It adds `iov_iter` support to the `/proc` filesystem, for example. [This patch](<https://lwn.net/ml/linux-kernel/20200903142242.925828-6-hch@lst.de/>) converts `kernel_read()` and `kernel_write()` (yet another way to perform I/O on kernel-space buffers) to `iov_iter`, removing the `set_fs()` calls previously used there. The [`splice()`](<https://man7.org/linux/man-pages/man2/splice.2.html>) implementation is changed in a way that might break existing users: it simply no longer works if the data source is a device that does not support the `splice_read()` method. Hellwig said that the affected users all appear to have working fallbacks in place, but that specific devices can gain `splice_read()` methods if the need turns out to exist. 

After a few more patches to remove the last uses of `set_fs()` from the x86 and PowerPC architectures, support for `set_fs()` itself is disabled and the task is complete. These patches are currently in linux-next, and thus should be merged for the 5.10 release. Hellwig has also posted [a patch set for RISC-V](<https://lwn.net/ml/linux-kernel/20200907055825.1917151-1-hch@lst.de/>), and Arnd Bergman has [a patch set for Arm](<https://lwn.net/ml/linux-kernel/20200918124624.1469673-1-arnd@arndb.de/>), but those have not yet been applied. Hellwig intends to work through the remaining architectures, removing `set_fs()` from each. 

The patches described above are only a small portion of the effort that has gone into making it possible to finally get rid of `set_fs()`. The end result of all this work is the near elimination of a kernel interface that has been deemed dangerous for almost as long as it has existed — and it has been around for a long time. It is an example of a form of kernel development that tends not to create headlines, but which quietly keeps the kernel maintainable in the long term. Tasks like this often suffer from a lack of attention, but they do tend to get done in the long run, which is a good thing; even after nearly 30 years, there is a lot of cleaning up still to be done in the kernel.
