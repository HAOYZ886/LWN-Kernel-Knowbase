---
title: Reconsidering BPF ABI stability
url: https://lwn.net/Articles/921088/
date: "January 26, 2023"
category: "BPF; Development model-User-space ABI"
author: "By Jonathan Corbet January 26, 2023"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
January 26, 2023

The BPF subsystem exposes many aspects of the kernel's internal algorithms and data structures; this naturally leads to concerns about maintaining interface stability as the kernel changes. The longstanding position that BPF offers no interface-stability guarantees to user space has always seemed a little questionable; kernel developers have, in the past, found themselves having to maintain interfaces that were not intended to be stable. Now the BPF community is starting to think about what it might mean to provide explicit stability promises for at least some of its interfaces. 

#### Hooks, helpers, and kfuncs

BPF allows programs loaded by user space to be attached to any of a large number of hooks and run within the kernel — after the subsystem's verifier concludes that those programs cannot harm the system. A program will gain access to the kernel data structures provided to it by the hook it is attached to. In some cases, the program can modify that data directly, thus directly affecting the operation of the kernel; in others, the kernel will act on the value returned by a BPF program to, for example, allow or disallow an operation. 

There are also two mechanisms by which the kernel can make additional functionality available to BPF programs. [Helper functions](<https://man7.org/linux/man-pages/man7/bpf-helpers.7.html>) (or "helpers") are special functions that are written for the purpose of being provided to BPF programs; they have been present since the beginning of the extended-BPF era. The mechanism known as [kfuncs](<https://www.kernel.org/doc/html/latest/bpf/kfuncs.html>) is newer; it allows _any_ kernel function to be made available to BPF, possibly with some restrictions applied. Kfuncs are simpler and more flexible; had they been implemented first, it seems unlikely that anybody would have added helpers later. That said, kfuncs have a significant limitation that they are only accessible to JIT-compiled BPF code, so they are unavailable on architectures lacking JIT support (a list that currently includes 32-bit Arm and RISC-V, though patches adding that support for both are in the works). 

Kfuncs are easily added and generally see little review outside of the small core-BPF community. Most kfuncs in existing kernels reach deeply into the networking subsystem, providing access for [congestion-control algorithms](<https://lwn.net/Articles/811631/>), the [express data path (XDP)](<https://lwn.net/Articles/750845/>), and more. But there are also kfuncs for access to the core `task_struct` structure, [crashing the kernel](<https://lwn.net/Articles/901284/>), access to control groups, read-copy-update, kernel linked lists, and more. The list of kfuncs seems to grow with each kernel release. 

Each kfunc makes some useful functionality available to BPF programs, but almost every one also exposes some aspect of how the kernel works internally. One cannot, for example, implement a congestion-control algorithm in BPF without significant visibility into how the networking subsystem works and the ability to affect that operation. That is (usually) fine until the kernel changes — which happens frequently. Within the kernel, API changes are a routine occurrence; developers simply fix all of the affected kernel code as needed. But those developers are unable to fix BPF code that may be widely deployed on production systems. Any changes to kernel code that has BPF hooks in it risks breaking an unknown amount of user-space code using those hooks. 

The normal rule in the kernel community is that changes cannot break user space; if a patch is found to have broken programs in actual use, that change will normally be reverted. User-space APIs are thus a significant constraint on kernel development; in the worst case, they could block needed changes entirely. That is a prospect that makes kernel developers nervous about providing BPF access to their subsystems. 

The intersection of BPF and interface stability has come up numerous times on the mailing lists and at conferences. The BPF community's position has always been clear: the interfaces used by BPF programs are analogous to those used by loadable kernel modules. They are thus a part of the internal kernel API rather than the user-space API and have no stability guarantees. If a kernel change breaks a BPF program, it is the BPF program that will have to adapt. 

It is a convenient position, but it's never been entirely clear that this position is tenable in the long term. If a kernel change breaks a BPF program that is widely used, there will be substantial pressure to revert that change, regardless of what the official position is. Consider, for example, [a human-interface-device (HID) driver](<https://lwn.net/Articles/909109/>) implemented in BPF. If this mechanism is successful, distributions will eventually ship BPF HID drivers, and users will likely not even know that they are using a BPF program. They are unlikely to be amused if, in response to a future kernel update that breaks their mouse, they are told that it is their fault for using internal kernel APIs. 

Beyond that, a lack of interface stability guarantees may well be an impediment to the future adoption of BPF by developers. It may come as a surprise to learn that developers tend not to be happy if they have to deal with bug reports because an interface they used was changed. There will be a strong incentive to avoid an API that is presented as being unstable, even if that API could be the path to a better solution for their problem. 

#### Documenting BPF interface stability

The BPF developers, it seems, have been talking about these problems; one tangible result from those discussions was [this documentation patch](<https://lwn.net/ml/bpf/20230117212731.442859-1-toke@redhat.com/>) recently posted by Toke Høiland-Jørgensen that described how a (partial) stability guarantee could work: 

> This patch adds a description of the (new) concept of "stable kfuncs", which are kfuncs that offer a "more stable" interface than what we have now, but is still not part of [the kernel's user-space API]. 
> 
> This is mostly meant as a straw man proposal to focus discussions around stability guarantees. From the discussion, it seemed clear that there were at least some people (myself included) who felt that there needs to be some way to export functionality that we consider "stable" (in the sense of "applications can rely on its continuing existence"). 

There are, he said in the cover letter, a couple of approaches that could be taken. One would be to declare that helper functions are a stable interface, and that kfuncs are not. Should a kfunc prove to be sufficiently useful that developers feel the need for a stability guarantee, the kfunc could be promoted to a helper. Alexei Starovoitov [objected](<https://lwn.net/ml/bpf/20230119043247.tktxsztjcr3ckbby@MacBook-Pro-6.local/>) to that idea, noting that the promotion would, itself, be an ABI break: 

> Say, we convert a kfunc to helper. Immediately the existing bpf prog that uses that kfunc will fail to load. That's the opposite of stability. We're going to require the developer to demonstrate the real world use of kfunc before promoting to stable, but with such 'promotion' we will break bpf progs. 

An alternative described by Høiland-Jørgensen is to explicitly mark kfuncs that are meant to be stable. All kfuncs now must be declared to the kernel with the [`BTF_ID_FLAGS()` macro](<https://www.kernel.org/doc/html/latest/bpf/kfuncs.html#annotating-kfuncs>), which takes a number of flags modifying that kfunc's treatment by the BPF subsystem. `KF_ACQUIRE`, for example, says that the function will return a pointer to a reference-counted object that must be released elsewhere in the program, while `KF_SLEEPABLE` says that the kfunc might sleep. A new flag, `KF_STABLE`, would be used to mark kfuncs that the kernel developers will go out of their way not to break. 

Even then, though, the document makes it clear that a `KF_STABLE` kfunc still lacks the same level of guarantee as the rest of the user-space ABI: ""Should a stable kfunc turn out to be no longer useful, the BPF community may decide to eventually remove it"". That removal would be preceded by a period in which the kfunc would be marked as being deprecated (with the new `KF_DEPRECATED` flag) that would generate a warning whenever a BPF program used it. 

Starovoitov (in the above-linked message) was fairly negative about this proposal. All kfuncs should be treated as if they were stable, he said, with the amount of effort that is justified in maintaining stability increasing as the use of the kfunc goes up. A strong stability guarantee would require an active developer community that is clearly making use of the kfunc: 

> The longer the kfunc is present the harder it will be for maintainers to justify removing it. The developers have to stick around and demonstrate that their kfunc is actually being used. The better developers do it the bigger the effort maintainers will put into keeping the kfunc perfectly intact. 

He also made the point that there are currently no kfuncs in the kernel that would merit the stable marking, because nobody has done any research into which kfuncs are actually in production use. Similarly, there are currently no deprecated kfuncs. Thus, he said: ""Introducing KF_STABLE and KF_DEPRECATED right now looks premature"". Høiland-Jørgensen [responded](<https://lwn.net/ml/bpf/875ycvo1im.fsf@toke.dk/>) that, at a minimum, the development community should promise not to remove any kfuncs without implementing a deprecation period first. 

David Vernet [was also unconvinced](<https://lwn.net/ml/bpf/Y8gV%2FXveyPmf8alQ@maniforge.lan/>) about the proposal. It would be better, he said, to put information about stability and deprecation into the kernel documentation rather than in the code. He also worried that `KF_STABLE` lacked the ability to express the types of changes that might come, and suggested that some sort of symbol-versioning mechanism might be better. 

One aspect of the problem that was _not_ touched on in the discussion was the fact that, as BPF reaches into more kernel subsystems, maintaining stability will require the cooperation of developers outside of the BPF community — developers who may never have signed onto any stability guarantee. If, for example, a future `task_struct` change ends up being blocked because it breaks some BPF program, the resulting fireworks would likely require a lot of popcorn to get through. To be truly effective, any promise of stability for kfuncs is probably going to require a wider discussion than has been seen so far. 

For all of these reasons, it seems unlikely that the scheme described in Høiland-Jørgensen's patch will be adopted in that form. Instead, the stability status of kfuncs may remain somewhat ambiguous, Starovoitov's statement that ""we need to finish it now and don't come back to it again every now and then"" notwithstanding. Stability guarantees are not something to be made lightly, so it is not surprising that the BPF community still seems to not want to rush into doing any such thing.
