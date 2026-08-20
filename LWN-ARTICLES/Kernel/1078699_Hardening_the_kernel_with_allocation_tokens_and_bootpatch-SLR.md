---
title: "Hardening the kernel with allocation tokens and bootpatch-SLR"
url: https://lwn.net/Articles/1078699/
date: "June 25, 2026"
category: "Memory management-Slab allocators; Security-Kernel hardening"
author: "By Jonathan Corbet June 25, 2026"
---

> **For humans, by humans**
> 
> Every article on LWN.net is written for humans, by humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the slop at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
June 25, 2026

There is a lot of work going into eliminating exploitable bugs from the kernel and preventing the addition of new ones. Even if this work is maximally successful, though, there is no chance that the kernel will be free of these bugs anytime soon. Thus, there is also ongoing interest in hardening the kernel to make the existing bugs more difficult to exploit. The upcoming 7.2 kernel release will include a change to how dynamically allocated structures are placed in memory to make them harder to overwrite, while a project to randomize structure layout at boot time has a rather longer timeline. 

#### Memory partitioning with allocation tokens

The kernel's slab allocator handles memory management for small objects. Allocations come from buckets of fixed-size (mostly powers of two) objects, and many types of objects with the same bucket size can be allocated from the same slab. This arrangement allows efficient use of memory, but the intermingling of object types in this way can make the kernel more vulnerable to certain types of exploits. If the kernel can be convinced to overrun an object of a given type, it may end up accessing or corrupting other objects with entirely unrelated types. Additionally, [heap-spraying attacks](<https://en.wikipedia.org/wiki/Heap_spraying>) may be used to fill much of the kernel's memory space with known data, which can make other sorts of bugs (such as an erroneous pointer dereference) easier to exploit. 

Back in 2023, [a set of patches](<https://lwn.net/Articles/938637/>) was merged in an attempt to harden the kernel against such attacks; this mechanism works by partitioning the memory used by the slab allocator to satisfy requests. Rather than use one set of slabs to provide (for example) 64-byte memory chunks, the allocator uses sixteen of them. The specific slab used for any given request is chosen randomly, but in such a way that the same slab is always used for allocations made at the same call site. In this way, the ability to fill memory with techniques like heap spraying is much reduced. The ability to overwrite a targeted object with a buffer overflow is also reduced; most object types will be located in distant slabs, and the specific mix of object types in any given slab will differ from one boot to the next. 

Partitioning that is randomized in this way makes many attacks harder, but it is a probabilistic defense. It may separate a target object from the vulnerable object most of the time, but it will still put them into the same slab one boot out of sixteen. In multi-system cloud deployments, some systems will certainly end up with an attacker-friendly placement of vulnerable objects. Naturally, security-oriented kernel developers would like to do better than that. 

One attempt to do better is [type-based slab partitioning](<https://lwn.net/ml/all/20260511200136.3201646-1-elver@google.com/>), by Marco Elver, which was [merged](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=feb662d9168b>) for the 7.2 release. As its name would suggest, it works by separating allocations based on the type of the object being allocated rather than the address of the allocation call site. 

To do so, this feature takes advantage of the [allocation tokens feature](<https://clang.llvm.org/docs/AllocToken.html>) that is available with the Clang 23 compiler. In short, this feature provides a new compiler built-in function, `__builtin_infer_alloc_token()`, which generates an integer token value from the type(s) of the argument(s) passed into it. That token, in turn, can be used to select the partition from which an object will be allocated. The result is that objects of the same type will always be allocated from the same partition, even if there are several call sites performing the allocations. A structure involved in a vulnerability will always be confined to a single partition, and will always be separated from most other structure types. 

As an additional hardening feature, the generation of the token takes into account whether the type(s) in question are structures that contain pointers; the space for tokens representing pointer-containing types is disjoint from the space for all other types. As a result, types that do not contain pointers are kept separate from those that do, making any overrun vulnerabilities with those types harder to exploit for the purpose of overwriting pointers. 

The potential downside of this approach is that the mapping from types to partitions is now deterministic and, thus, predictable by attackers. The older, randomized partitioning is still available as a configuration option for those who prefer it. Distributors will have to make a choice, though, regarding which hardening variant (if either) they wish to enable for their users. 

#### Run-time structure-layout randomization

Attackers can benefit from any information they have about how the memory in a given system is laid out. That includes the layout of fields within structures; without that information, targets for attack are harder to locate in a running system. Structures are laid out in the source, and the C standard is fairly clear about how a structure declaration should map to the resulting representation in memory. Attackers will use that information, but may be thrown off if the in-memory layout differs from the source in unpredictable ways. 

This not a new idea; the kernel has had support for [the "randstruct" GCC plugin](<https://lwn.net/Articles/722293/>), which randomizes the order of fields within structures, for years. Randstruct, though, is a compile-time operation; it rearranges structure fields while the kernel is being built. For a site running a custom kernel, build-time randomization may be sufficient. For kernels shipped by distributors, though, it is not particularly helpful; an attacker can look at the kernel image and know how a structure will be laid out in all systems running that particular kernel. 

York Jasper Niebuhr recently showed up with [an early-stage patch set](<https://lwn.net/ml/all/20260620133222.94647-1-yjn@yjn-systems.com>), called Bootpatch-SLR, that is meant to address that shortcoming. It performs structure randomization at boot time, meaning that every kernel will have a different layout, and that layout will change every time the system is rebooted. That should make life harder for anybody who is trying to target a specific structure field. 

Bootpatch-SLR works by annotating structures to be randomized in the source with a set of special macros; [this patch](<https://lwn.net/ml/all/20260605202511.79272-6-yjn@yjn-systems.com>) marking up `struct task_struct` shows how it works. In short, a set of fields to be randomized is annotated with markers like: 

```
struct foo {
            spslr_struct_fields_start
    	/* fields to be randomized go here */
    	spslr_struct_fields_end
        };
```

These markers turn into special ELF sections that will be included in the resulting kernel binary; they mark not only the fields to rearrange, but also all of the references to those fields. When the kernel boots, it reads those sections, rearranges the indicated structure fields and, by way of a bunch of magic code-patching, updates all of the references to match. 

Naturally, there are places where fields cannot be so easily rearranged. Many places in the kernel embed one structure as the first element of a containing structure, then assume that a pointer to the container can be cast to a pointer to the embedded structure. Relocating the embedded structure is likely to produce results that are rather more random than desired. The layout of other structures may be determined by protocol specifications or hardware. Structures that should not be changed at all can simply be left unannotated; others, though, may be able to tolerate the reordering of some fields while others are left untouched. For that case, there is a special marker, `__spslr_field_fixed`, that can be used to mark fields that need to keep their declared position within the structure. 

This idea is appealing; once fully implemented, it could significantly increase the resistance of existing kernels to attack. That is a rather distant goal at the moment, though. Among other things, Bootpatch-SLR is implemented as a GCC plugin, and [the long-term plan](<https://lwn.net/Articles/851090/>) is to remove support for these plugins from the kernel's build system. An attempt to add a new one now is unlikely to be successful. There are a number of other details that need to be worked out as well; several of them are described in [the "known issues" section](<https://spslr.yjn-systems.com/bootpatch-slr.html#:~:text=Known%20issues,-The>) of the extensive [online documentation for Bootpatch-SLR](<https://spslr.yjn-systems.com/bootpatch-slr.html>). 

Time will tell whether Bootpatch-SLR will reach a point where it can be merged into the mainline kernel; history is full of bright ideas and interesting prototypes that never reached that point. If it does get there, though, the security benefits could be significant.
