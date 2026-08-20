---
title: Yet another try at the BPF program allocator
url: https://lwn.net/Articles/915005/
date: "November 28, 2022"
category: "BPF-Memory management"
author: "By Jonathan Corbet November 28, 2022"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
November 28, 2022

The BPF subsystem, which allows code to be loaded into the kernel from user space and safely executed in the kernel context, is bound to create a number of challenges for the kernel as a whole. One might not think that allocating memory for BPF programs would be high on the list of problems, but life (and memory management) can be surprising. The attempts to do a better job of providing space for compiled BPF code have, to date, only been partially successful; now Song Liu is back with [a new approach](<https://lwn.net/ml/linux-mm/20221107223921.3451913-1-song@kernel.org/>) to finish the job. 

#### Small, transient, and numerous

The problem with BPF programs is that they tend to be small, are often transient, and can be numerous. That, alone, would not be hard for the kernel to deal with; the slab allocators are highly tuned for the efficient allocation and freeing of small objects. But BPF programs, being executable code, must be stored in memory that allows execute access, and that complicates the picture. 

Any memory that is both executable and writable presents an attractive target for attackers, so the kernel goes well out of its way to prevent that combination from occurring; some architectures prohibit it entirely. The kernel's own code is loaded at boot time, made read-only, and (usually) never changed again. Loadable modules, which require the addition of kernel code at run time, complicate things a bit, but modules are relatively large and tend to be stable. The kernel will load the modules it needs shortly after boot, and the set of loaded modules will rarely change thereafter. As a result, even if the handling of loadable modules is not optimal, things normally work well enough anyway. 

As noted above, though, BPF programs can come and go frequently, and there can be a lot of small programs in the system. All of this would be fine in the absence of the prohibition on memory that is both writable and executable; that restriction requires that memory holding BPF programs, which are executable, be made non-writable. That, in turn, requires changing the permissions in the "direct map", the range of kernel address space that (on 64-bit systems) maps all of the system's physical memory. Even if direct-map addresses are not used to access BPF memory (as would happen if the `vmalloc()` family of allocators is used to obtain it), the existence of a writable direct mapping to executable code would create a potential vulnerability. 

The kernel's direct map uses huge-page mappings (of 1GB size when possible). Huge-page mappings reduce the pressure on the system's translation lookaside buffer (TLB) and improve the performance of the system overall. If a portion of the direct map must be made read-only, though, then the huge page the contains it must be split into smaller pages, fragmenting the direct map with a measurable impact on performance. Doing that once or twice might not be a big problem but, in a system where BPF programs come and go frequently, the impact on the direct map can be severe. 

The smallness of BPF programs also turns out to be a problem. In older kernels, each BPF program was loaded into its own (4KB) page, meaning that, often, most of the page was wasted. If many of these programs are loaded, that wasted memory starts to add up. 

In February, Liu [set out to solve these problems](<https://lwn.net/Articles/883454/>). The "bpf_prog_pack" allocator worked by allocating 2MB huge pages from the kernel, then handing out portions of those pages for BPF programs as they are loaded. The concentration of multiple BPF programs into huge pages addressed both problems: it minimized fragmentation of the direct map and reduced memory waste by packing BPF programs together in the same page. This allocator looked like a good solution and was quickly pulled into the mainline during the 5.18 merge window. 

Unfortunately, [a number of problems](<https://lwn.net/Articles/892743/>) quickly surfaced, and much of the bpf_prog_pack functionality was backed out despite the fact that the source of some of the trouble was to be found in the memory-management subsystem. The allocator is still present in the kernel, but it uses 4KB "base" pages, so it does not help performance as much as it could. 

#### Trying again

Liu's new proposal replaces bpf_prog_pack with a new allocator that addresses the complaints about the previous version and, once again, uses huge pages to hold BPF programs. That leads to improved performance: 

> Based on our experiments, we measured 0.5% performance improvement from bpf_prog_pack. This patchset further boosts the improvement to 0.7%. The difference is because bpf_prog_pack uses 512x 4kB pages instead of 1x 2MB page. 

The use of 2MB pages is now possible as the result of fixing the related problems in the memory-management subsystem. This new allocator goes beyond the use of huge pages, though, and creates a new API for the management of transient, executable code in the kernel: 

```
void *execmem_alloc(unsigned long size, unsigned long align);
        void *execmem_fill(void *dst, void *src, size_t len);
        void execmem_free(void *addr);
```

Any kernel subsystem that needs to set up a segment of executable code can allocate the memory with `execmem_alloc()`. The memory that is returned will have read-only protection, so the caller cannot copy the code into it directly. Instead, `execmem_fill()` must be called to populate this memory with the executable text. On the x86 architecture (the only one that supports this mechanism now), the "text_poke" machinery will be used to safely copy the code while dodging the many race conditions that can present themselves when code is being modified. If a range of executable memory is no longer needed, it can be returned with `execmem_free()`. 

The advantage of this new API is that it is not limited to just BPF programs; it can also potentially be used in other places where code is loaded into the kernel — specifically for loadable modules. That would improve the efficiency of those allocations while simultaneously reducing the number of code-loading implementations in the kernel. That seems like a significant benefit, but there is just one little problem: the module loader has not been changed to actually use this API, so there is no proof that it will work in that context. 

Indeed, it almost certainly will not work for module loading yet, simply because there is no support for any architectures other than x86. Loading code into a running kernel is a tricky business, and the details of how it can be done safely vary widely from one architecture to the next. A number of architectures now implement at least parts of the text_poke API, which simplifies the task, but text_poke is not universal; arm64 does not support it, for example. Architectures also have differing requirements around the placement of data areas for modules; it may not work to put a module's BSS memory far away from its text, for example. All of this adds up to a number of potential headaches for anybody trying to actually use the new API for module loading. 

Reviewers of this work would, understandably, like some assurance that the new API can work beyond BPF before accepting it; Mike Rapoport, for example, has [asked](<https://lwn.net/ml/linux-mm/Y2o9Iz30A3Nruqs4@kernel.org/>) for ""at least some explanation how modules etc can use execmem_ APIs without breaking !x86 architectures"". Rick Edgecombe [responded](<https://lwn.net/ml/linux-mm/9e59a4e8b6f071cf380b9843cdf1e9160f798255.camel@intel.com/>) with an assertion that other architectures could be supported with minor changes to the API, but questioned whether it it is truly necessary to solve the whole problem at this point. 

Luis Chamberlain has also [expressed frustration](<https://lwn.net/ml/linux-mm/Y3P%2F9DXAjKhmoIvm@bombadil.infradead.org/>) at the lack of solid (and reproducible) data showing how this work improves system performance. He clearly sees some advantages overall, though, since one of his [complaints](<https://lwn.net/ml/linux-mm/Y2mXI1WHuhRW7Jt+@bombadil.infradead.org/>) is that the patch changelogs do not sufficiently highlight ""the gains of this effort helping with the long term advantage of centralizing the semantics for permissions on memory"". Liu has [responded](<https://lwn.net/ml/linux-mm/CAPhsuW4_aYvPJUfCBkMygKPpHx7Y3xPCV7ewLGGAhyztJq3dhA@mail.gmail.com/>) with a bit more data on TLB-miss improvement. 

The benefits of the work seem clear, should it manage to not run into surprises like its predecessor. The biggest question with regard to merging would seem to be just how much work will be required to convince reviewers that this API can handle the module case. If a complete solution is required, the new BPF program allocator seems unlikely to land anytime soon. Since there are no user-space API issues to resolve, though, it should be possible to proceed with the BPF solution once reviewers are convinced that it does not actively lead in the wrong direction.
