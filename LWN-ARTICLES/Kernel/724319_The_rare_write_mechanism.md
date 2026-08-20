---
title: "The \"rare write\" mechanism"
url: https://lwn.net/Articles/724319/
date: "June 1, 2017"
category: "Security-Kernel hardening"
author: "June 1, 2017 This article was contributed by Nur Hussein"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

June 1, 2017

This article was contributed by Nur Hussein

One of the ways to harden the kernel is by tightening permissions on memory to write-protect as much run-time data as possible. This means the kernel makes some data structures read-only to prevent malicious or accidental corruption. However, inevitably, most data structures need read/write access at some point. Because of this, a blanket read-only policy for these structures wouldn't work. Therefore, we need a mechanism that keeps sensitive data structures read-only when "at rest", but allows writes when the need arises. 

Kees Cook [proposed](<http://marc.info/?l=linux-kernel&m=149081147926918&w=2>) such a mechanism based on similar functionality that exists in the PaX/grsecurity patch set. Cook calls it "rare write", but that is a bit of misnomer considering that write access to some of the target data structures is not always rare. The resulting discussion on the Linux kernel mailing list fleshed out how the mechanism could be implemented, from a standpoint of both architecture-neutrality and performance.

Cook's proposed series of eleven patches for the `rare_write()` infrastructure consists of sample implementations for x86 and ARM, plus usage examples. Two new kernel configuration options, `HAVE_ARCH_RARE_WRITE` and `HAVE_ARCH_RARE_WRITE_MEMCPY`, were [added](<http://marc.info/?l=linux-kernel&m=149081147926916&w=2>) to let architectures define whether or not they have implemented the new mechanism. Cook's implementation contains architecture-specific code that relies on CPU features on x86 and ARM that selectively enable and disable write access to areas of memory. Thus, the data write routines cannot be preemptible, as interrupting the kernel after it enables writing on read-only memory would result in leaving a window of vulnerability open. Cook noted that his code is inlined to discourage [return-oriented programming](<https://en.wikipedia.org/wiki/Return-oriented_programming>) attacks on the write-enable routines, which would otherwise be a juicy target for such attacks.

The architecture-specific nature of the code became a point of contention in the resulting discussion on the linux-kernel and kernel-hardening mailing lists. But first, let's look at how Cook's proposed mechanism works. 

The kernel stores data in different ELF sections depending on how that data will be accessed: `.data` or `.bss` for normal, read-write data, in `.rodata` for read-only data, or in `.data..ro_after_init` for data that will [become read-only after initialization](<https://lwn.net/Articles/666550/>). Cook's newly introduced `__wr_rare` annotation, which for now is just a `#define` alias for `__ro_after_init`, would be used to mark data structures to be converted to read-only. Write access would be enabled via a new `rare_write()` macro, or in a critical section protected by `rare_write_begin()` and `rare_write_end()` macros. Cook gave a simple [example](<http://marc.info/?l=linux-kernel&m=149081147926919&w=2>) of the usage of the single `rare_write()` call by converting a function in `net/core/sock_diag.c` from:

```
sock_diag_handlers[family] = NULL;
```

To: 

```
rare_write(sock_diag_handlers[family], NULL);
```

There are also helper functions [to deal with linked lists](<http://marc.info/?l=linux-kernel&m=149081159026970&w=2>) that utilize the `rare_write_begin()` and `rare_write_end()` macro pair internally. 

#### Implementation

On x86 CPUs, read-only data is mapped onto pages marked with the read-only bit set in their page-table entries. CR0 is a control register on the x86 CPUs, and toggling the write-protect bit (bit 16) of CR0 allows this read-only protection to be either enabled or disabled. When the write-protect bit has been cleared on a CPU, any data, regardless of the page-table permissions, can be written by the kernel when running on that CPU. After some back and forth discussion, Cook's [implementation](<http://marc.info/?l=linux-arm-kernel&m=149143768912131&w=2>) of the rare write functions on x86 became the following: 

```
static __always_inline unsigned long __arch_rare_write_begin(void)
        {
               unsigned long cr0;
    
               cr0 = read_cr0() ^ X86_CR0_WP;
               BUG_ON(cr0 & X86_CR0_WP);
               write_cr0(cr0);
               return cr0 ^ X86_CR0_WP;
        }
    
        static __always_inline unsigned long __arch_rare_write_end(void)
        {
               unsigned long cr0;
    
               cr0 = read_cr0() ^ X86_CR0_WP;
               BUG_ON(!(cr0 & X86_CR0_WP));
               write_cr0(cr0);
               return cr0 ^ X86_CR0_WP;
        }
```

Mathias Krause [raised a concern](<http://marc.info/?l=linux-arm-kernel&m=149155411211517&w=2>) that non-maskable interrupts would still be able to interrupt the CPU inside the critical section, and questioned ~~if PaX's implementation that Cook's patch was based on~~ whether the implementation was indeed correct. Thomas Gleixner [suggested](<http://marc.info/?l=linux-kernel&m=149155839912794&w=2>) that the proper way to handle writing to a read-only memory location is to create a writable shadow mapping to that physical memory location, and update the value of the data in that location through the shadow map. This approach would not require clearing the CR0 write-protect bit which, he suggested, is too dangerous to ever do. Gleixner provided this piece of pseudocode to illustrate his idea: 

```
write_rare(ptr, val)
        {
                mp = map_shadow_rw(ptr);
                *mp = val;
                unmap_shadow_rw(mp);
        }
```

He added:

map_shadow_rw() is essentially the same thing as we do in the highmem case where the kernel creates a shadow mapping of the user space pages via kmap_atomic().

It's valid (at least on x86) to have a shadow map with the same page attributes but write enabled. That does not require any fixups of CR0 and just works.

After subsequent discussions on which method would be better with regard to performance, Andy Lutomirski [suggested](<http://marc.info/?l=linux-kernel&m=149158172821109&w=2>) setting up a separate `mm_struct` that had a writable alias of the target read-only data and using `use_mm()` to access it. Lutomirski argued this was more architecture-neutral and, if anyone was inclined to test its performance, they could do so against the CR0-toggle method. While Mark Rutland [agreed with this approach](<http://marc.info/?l=linux-kernel&m=149158221321267&w=2>), Gleixner [wasn't too sure](<http://marc.info/?l=linux-kernel&m=149159790526230&w=2>) if this was efficient. Cook [replied](<http://marc.info/?l=linux-kernel&m=149160005926868&w=2>) that there was a need for performance and efficiency, as this mechanism will be used for data structures that need extra protection, and not just those are are written to rarely: 

I probably chose the wrong name for this feature (write rarely). That's _usually_ true, but "sensitive_write()" was getting rather long. The things that we need to protect with this are certainly stuff that doesn't get much writing, but some things are just plain sensitive (like page tables) and we should still try to be as fast as possible with them.

PaX Team [responded](<http://marc.info/?l=linux-kernel&m=149176959725535&w=2>) that the weakness of the `use_mm()` approach is that it wouldn't scale, and that you couldn't use the approach inside `use_mm()` and `switch_mm()` themselves. These limitations [also concerned](<http://marc.info/?l=linux-kernel&m=149185519506015&w=2>) Cook: 

These are the limitations that concern me: what will we NOT be able to make read-only as a result of the use_mm() design choice? My RFC series included a simple case and a constify case, but I did not include things like making page tables read-only, etc. 

Lutomirski [replied](<http://marc.info/?l=linux-kernel&m=149185545706083&w=2>) that page table writes aren't rare, so there may need to be ""multiple levels of rareness"" to accommodate this. 

Cook also included an implementation for ARM-based CPUs. The ARM architecture organizes memory into regions called "domains", and each domain can have different access permissions. Cook [created](<http://marc.info/?l=linux-kernel&m=149081159126973&w=2>) a domain called `DOMAIN_WR_RARE` to store read-only data; whenever `rare_write()` is invoked on ARM, the code will [modify](<http://marc.info/?l=linux-kernel&m=149081159026972&w=2>) the domain permissions of `DOMAIN_WR_RARE` to enable writes, and disable them again after it is done.

#### Conclusion

After the various comments that were posted on the mailing list, Cook is working on another iteration of his patch set. An open question is how something like this can be implemented on architectures that do not have architecture support for enabling or disabling writing to read-only memory. Earlier, in his first RFC series on the kernel-hardening list, Cook [stated](<http://www.openwall.com/lists/kernel-hardening/2017/02/28/9>) that he isn't sure how to handle other architectures that do not have the requisite hardware support, but he does plan to get a "viable interface" for the architectures that do. Perhaps the way forward is to use the suggested method of writing to read-only data sections via shadow mappings, which can be implemented with `kmap_atomic()` as proposed by Gleixner.
