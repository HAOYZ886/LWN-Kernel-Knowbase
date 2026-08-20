---
title: Hardened usercopy
url: https://lwn.net/Articles/695991/
date: "August 3, 2016"
category: "Security-Kernel hardening; copy  user"
author: "By Jake Edge August 3, 2016"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
August 3, 2016

The kernel often copies data from and to user space, which makes `copy_to_user()` and `copy_from_user()` (and friends) rather frequently used kernel functions. But if the kernel can be tricked into copying too much data in either direction, security vulnerabilities can be the result. Long ago, grsecurity added the PAX_USERCOPY feature (created by the PaX team) to harden those calls, so that even poorly written code elsewhere in the kernel cannot truly copy more than it should. Code based on PAX_USERCOPY is now being proposed for inclusion into the mainline kernel. 

Kees Cook [posted](<https://lwn.net/Articles/693745/>) the first version of his "hardened usercopy" patches in early July. The patches are based on some earlier work that Casey Schaufler had done to port the PAX_USERCOPY feature from grsecurity to the mainline. Essentially, it tries to ensure that address ranges used to copy data to and from user space are valid. Cook is also working on patches for two other parts of the PAX_USERCOPY feature; this piece is configured into the kernel with the `CONFIG_HARDENED_USERCOPY` option. 

The main problems that can result from an errant user-space copy are either that too much data is copied to user space, resulting in leaking the contents of kernel memory, or that too much data is copied from user space, which can overwrite kernel memory. If an attacker can influence the allocation of objects on the kernel's heap and then overwrite some of those objects, they may be able to escalate privileges, run arbitrary code, or crash the kernel. Information leaks are generally less dangerous, but the kernel does have critical data (e.g. keys) that could be exposed. Beyond that, determining the layout of kernel memory by way of an information leak can also provide information needed to exploit other kernel flaws. 

The patches add several tests of the arguments to the `copy_*_user()` functions, which have the following prototypes: 

```
long copy_from_user(void *to, const void __user * from, unsigned long n);
        long copy_to_user(void __user *to, const void *from, unsigned long n);
```

Each call involves a user-space pointer and a kernel-space pointer; the user-space pointers are already checked in current kernels, so the patches only add tests for the kernel-space pointers. Those tests ensure that the address range doesn't wrap past the end of memory, that the kernel-space pointer is not null, and that it does not point to a zero-length `kmalloc()` allocation (i.e. `ZERO_OR_NULL_PTR()` is false). Also, if the address range overlaps the kernel text (code) segment, it is rejected. 

Beyond that, if the kernel-space address points into an object that has been allocated from the slab allocator, the patches ensure that what is being copied fits within the size of the object allocated. This check is performed by calling `PageSlab()` on the kernel address to see if it lies within a page that is handled by the slab allocator; it then calls an allocator-specific routine to determine whether the amount of data to be copied is fully within an allocated object. If the address range is not handled by the slab allocator, the patches will test that it is either within a single or compound page and that it does not span independently allocated pages. 

In addition, for copies involving the stack, the copied range must fit within the current process's stack. If there is architecture support for identifying stack frames, the copied range must fit within a single frame. 

In all cases, an address range that fails the tests will generate a log message with the pertinent information. It will also call `BUG()` to generate a kernel oops and kill the current process (i.e. the one that was trying to exploit a kernel hole of some kind). 

The patch set is broken up into three logical chunks: the main patch that adds the tests, patches that enable the feature for specific architectures (originally, x86, arm, arm64, ia64, powerpc, and sparc, with s390 added in a more recent patch set), and two patches that add heap-checking support for the SLAB and SLUB allocators. Cook noted that the SLOB allocator support in grsecurity ""seems entirely broken"", so he focused on testing SLAB and SLUB. In addition, stack frame checking has only been implemented for x86. 

Cook said that he ""couldn't detect a measurable performance change with these features enabled"", when running tests like kernel builds and [hackbench](<http://manpages.ubuntu.com/manpages/xenial/man8/hackbench.8.html>). That suggested that the feature could be turned on by default at some point, though it is turned off by default for now. Ingo Molnar [suggested](<https://lwn.net/Articles/696051/>) running a system-call-heavy workload to see if that had any measurable performance degradation, as he would also like to see the feature on by default. Linus Torvalds [said](<https://lwn.net/Articles/696052/>) that a `stat()`-heavy workload (e.g. something like `git diff`) would be one way to test it, but indicated that he thought the checks would not be all that onerous. 

Andy Lutomirski [wondered](<https://lwn.net/Articles/696055/>) if some of the infrastructure to validate the objects being copied should be given a different name, since it might be extended to more than just "usercopy" down the road. That set off a bit of a squabble between Molnar and PaX Team about the feature, threat models, and "bikeshedding". Cook, however, successfully [tamped down](<https://lwn.net/Articles/696056/>) the flickering flames: 

There's a long history of misunderstanding and miscommunication (intentional or otherwise) by everyone on these topics. I'd love it if we can just side-step all of it, and try to stick as closely to the technical discussions as possible. Everyone involved in these discussions wants better security, even if we go about it in different ways. If anyone finds themselves feeling insulted, just try to let it go, and focus on the places where we can find productive common ground, remembering that any fighting just distracts from the more important issues at hand. 

The patch set is in its [fourth revision](<https://lwn.net/Articles/696075/>) at this point; Cook has [requested](<https://lwn.net/Articles/696076/>) that it be pulled for 4.8. In the review process, some bugs have been fixed (notably some arm64 fixes and additions from Laura Abbott) and changes made, but no fundamental disagreement with the feature has emerged. As of this writing, the patches have not been pulled, but there were some prerequisites so it may simply be that Torvalds just hasn't gotten to it yet. But, if not for 4.8, it seems likely that we will see the feature appear in the mainline fairly soon.
