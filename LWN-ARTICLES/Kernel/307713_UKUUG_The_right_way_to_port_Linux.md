---
title: "UKUUG: The right way to port Linux"
url: https://lwn.net/Articles/307713/
date: "November 19, 2008"
category: Architectures
author: "By Jake Edge November 19, 2008"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
November 19, 2008

Arnd Bergmann pulled double duty at the recent [UKUUG Linux 2008 conference](<http://www.ukuug.org/events/linux2008/>) by giving a talk on each day of the event. His talk on Saturday, entitled "Porting Linux to a new architecture, the right way", looked at various problems with recent architecture ports along with a project he has been working on to simplify that process. By creating a generic template for architectures, some of the mistakes of the past can be avoided. 

This is one of Bergmann's pet projects, that ""I like to do for fun, when I am hacking on the kernel, but not for IBM"". The project and talk were inspired by a few new architectures that were merged--or were submitted for merging--in the last few years. In particular, the Blackfin and MicroBlaze architectures were inspiring, with the latter architecture still not merged, perhaps due to Bergmann's comments. He is hoping to help that situation get better. 

The biggest problem with architecture ports tends to be code duplication because people start by copying all of the files from an existing architecture. In addition, ""most people who don't know what they are doing copy from x86, which in my opinion is a big mistake"". According to Bergmann, architecture porters seem to ""first copy the header files and then change the whitespace"", which makes it difficult to immediately spot duplicated code. 

He points to `termbits.h` as an example of an include file that is duplicated in multiple architectures unnecessarily as the code is the same in most cases. He also notes there is ""incorrect code duplication"", pointing to new architectures that implement the `sys_ipc()` system call, resulting in ""brand new architectures supporting a broken interface for x86 UNIX from the 80s"". That call is a de-multiplexer for System V IPC calls that has the comment--dutifully duplicated into other architectures--"This is really horribly ugly". 

Then there are problems with ""code duplication by clueless people"" which includes a `sembuf.h` implementation that puts the padding in the wrong place because of 64 vs. 32-bit confusion. In addition, because code is duplicated in multiple locations, bug fixes that are made for one architecture don't propagate to all the places that need the fix. As an example he noted a bug fix made by Sparc maintainer David Miller in the x86 tree that didn't make it into the Sparc tree. Finally, there are ABIs that are being needlessly propagated in new architecture ports: system calls that are implemented in terms of newer calls are still present in new ports even though it could all be handled in libc. 

The "obvious" solution is to create a generic architecture implementation that can be used as a starting point for new ports. Bergmann has been working on that, resulting in a 3000 line patch that ""should make it very easy for people to port to new architectures"". To start with, it defines a canonical ABI that is a list of all of the system calls that need to be implemented for a new architecture. It puts all of the required include files into the `asm-generic` directory that new ports can just include--or copy if they need to modify them. 

Unfortunately, things are not quite that simple of course, there are a number of problem areas. There are ""lots of things you simply cannot do in a generic way"". Most of these things are fairly hardware-specific areas like MMU support, atomics, interrupts, task switching, byte order, signal contexts, hardware probing and the like. 

Bergmann decided to go ahead by defining away some of these problems in his `example` architecture. So, there is no SMP or MMU support with the `asm-generic/atomic.h` and `asm-generic/mmu_context.h` include files being appropriately modified. Many of the architecture-specific functions have been stubbed out in `arch/example/kernel/dummy.c` so that he can compile the template architecture. 

The `example` architecture uses an Open Firmware device tree to describe the hardware that is available at boot time. Open Firmware ""is a bit like what you have with the new Intel EFI firmware, but it's a lot nicer"". A flattened device tree data structure is passed to the kernel at boot time by the bootloader, so Bergmann will be able make it to the next step: making it boot. 

As one might guess, there is still more work to be done. There are eight header files that are needed from the `asm-example` directory, but Bergmann hopes to reduce that some. He notes that there are other architecture-specific areas that need work. For example, every single architecture has its own implementation of TCP checksums in assembly language, which may not be optimal 

Bergmann pointed attendees at the `ukuug2008` branch of his kernel.org playground git tree: [git://git.kernel.org/pub/scm/linux/kernel/git/arnd/playground.git](<http://git.kernel.org/?p=linux/kernel/git/arnd/playground.git;a=shortlog;h=ukuug2008>) to see the current state of his example architecture. It looks to be a nice addition to the kernel that will likely result in better architecture ports down the road.
