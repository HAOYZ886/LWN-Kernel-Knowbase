---
title: The road to Zettalinux
url: https://lwn.net/Articles/908026/
date: "September 16, 2022"
category: Architectures
author: "By Jonathan Corbet September 16, 2022 LPC"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
September 16, 2022

* * *

[LPC](<https://lwn.net/Archives/ConferenceByYear/#2022-Linux_Plumbers_Conference>)

Nobody should need more memory than a 64-bit pointer can address — or so developers tend to think. The range covered by a pointer of that size seems to be nearly infinite. During the Kernel Summit track at the [2022 Linux Plumbers Conference](<https://lpc.events/>), Matthew Wilcox took the stage to make the point that 64 bits may turn out to be too few — and sooner than we think. It is not too early to start planning for 128-bit Linux systems, which he termed "ZettaLinux", and we don't want to find ourselves wishing we'd started sooner. 

The old-timers in the audience, he said, are likely to have painful memories of the 32-bit to 64-bit transition, which happened in the mid-1990s. The driving factor at the time was file sizes, which were growing beyond the 2GB that could be represented with signed, 32-bit numbers. The Large File Summit in 1995 worked out [the mechanisms](<https://www.opengroup.org/platform/lfs.html>) ("lots of dreadful things") for handling larger files. Developers had to add the new `lloff_t` size for 64-bit file sizes and the [`llseek()` system call](<https://man7.org/linux/man-pages/man2/llseek.2.html>) to move around in large files. Wilcox said that he would really prefer not to see an `lllseek()` for 128-bit offsets. 

[![\[Matthew Wilcox\]](https://static.lwn.net/images/conf/2022/lpc/MatthewWilcox-sm.png)](<https://lwn.net/Articles/908089/>) Similarly, he does not want to see the equivalent of `CONFIG_HIGHMEM` on 128-bit systems. The ["high memory" concept](<https://lwn.net/Articles/813201/>) was created to support (relatively) large amounts of memory on 32-bit systems. The inability to address all of physical memory with a 32-bit address means that the kernel had to explicitly map memory before accessing it and unmap it afterward. Vendors are still shipping a few systems that need high memory, but it only represents a cost for 64-bit machines. Linux should transition to 128-bits before the 64-bit limitation falls behind memory sizes and forces us to recreate high memory. 

The solution, he said, is to move to CPUs with 128-bit registers. Processors back to the Pentium series have supported registers of that size, but they are special-purpose registers, not the general-purpose registers we will need. Looking at industry projections, Wilcox estimated that we would need 128-bit file-size values around 2040; he would like to see operating-system support for that in place by 2035. Address spaces are likely to grow beyond 64 bits around 2035 as well, so everything seems to be converging on that date. 

That said, he has talked with security-oriented developers who say that 2035 is far too late; 128-bit pointers are needed now. Address-space layout randomization, by changing the placement of objects in the virtual address space, is essentially using address-space bits for security; the more bits it has, the more effective that security is. When huge pages are in use, the number of available bits is low; 128-bit pointers would be helpful here. Similarly, technologies like [linear address masking](<https://lwn.net/Articles/902094/>) and [memory tagging](<https://lwn.net/Articles/834289/>) need more address bits to be truly effective. The experimental [CHERI architecture](<https://www.cl.cam.ac.uk/research/security/ctsrd/cheri/>) is using 12**9** -bit pointers now. 

How would this look in the kernel? Wilcox had originally thought that, on a 128-bit system, an `int` should be 32 bits, `long` would be 64 bits, and both `long long` and pointer types would be 128 bits. But that runs afoul of deeply rooted assumptions in the kernel that `long` has the same size as the CPU's registers, and that `long` can also hold a pointer value. The conclusion is that `long` must be a 128-bit type. 

The problem now is that there is no 64-bit type in the mix. One solution might be to "ask the compiler folks" to provide a `__int64_t` type. But a better solution might just be to switch to Rust types, where `i32` is a 32-bit, signed integer, while `u128` would be unsigned and 128 bits. This convention is close to what the kernel uses already internally, though a switch from "`s`" to "`i`" for signed types would be necessary. Rust has all the types we need, he said, it would be best to just switch to them. 

The system-call ABI is going to need thought as well. There are a lot of 64-bit pointer values passed between the kernel and user space in the current ABI. Wilcox suggested defining a new `__ptr_t` type to hold pointers at the user-space boundary; he said it should be sized at 256 bits. That's more than the 128 bits needed now, but gives room for any surprising future needs, and "it's only memory" in the end. 

Another problem is that, currently, the kernel only supports one compatibility personality, which is most often used to run 32-bit binaries on 64-bit systems. That will need to be fixed to be able to support both 32-bit and 64-bit applications on a 128-bit kernel. There are also many places in the kernel that are explicitly checking whether integers are 64-bits wide; those will all need to be tracked down and fixed to handle the 128-bit case too. 

All this sounds like a lot of work, he said, but in the end it's just porting Linux to a new architecture, and that has been done many times before. 

Ben Herrenschmidt said that, if he were in Wilcox's shoes, he would automate the generation of the compatibility definitions to minimize potential problems going forward. Wilcox answered: "In _my_ shoes?". His next slide, labeled "next steps", started with the need to find somebody to lead this effort. He said he would do it if he had to, but would rather somebody else did it. His hope was that Arnd Bergmann would step up to the task, "not that I don't like Arnd". Other steps include actually getting a 128-bit system to develop on; there is currently the beginning of a 128-bit extension defined for RISC-V that could be a starting point, probably via QEMU emulation initially. 

Bergmann appeared briefly on the remote feed to point out that the problem can be broken into two parts: running the kernel with 128-bit pointers, and supporting a 128-bit user space. He suggested starting by just supporting the user-space side while keeping the kernel at 64 bits as a way to simplify the problem. Wilcox said he hadn't thought of that, but that it could be an interesting approach. Whichever approach is taken, he concluded, the community should get started to avoid repeating the most painful parts of the 64-bit transition. There is, he said, still time to get the job done. 

[Thanks to LWN subscribers for supporting my travel to this event.]
