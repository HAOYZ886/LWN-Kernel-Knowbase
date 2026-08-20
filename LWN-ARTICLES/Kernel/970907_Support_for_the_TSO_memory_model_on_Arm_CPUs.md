---
title: Support for the TSO memory model on Arm CPUs
url: https://lwn.net/Articles/970907/
date: "April 26, 2024"
category: "Architectures-Arm; Memory model"
author: "By Jonathan Corbet April 26, 2024"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
April 26, 2024

At the CPU level, a memory model describes, among other things, the amount of freedom the processor has to reorder memory operations. If low-level code does not take the memory model into account, unpleasant surprises are likely to follow. Naturally, different CPUs offer different memory models, complicating the portability of certain types of concurrent software. To make life easier, some Arm CPUs offer the ability to emulate the x86 memory model, but efforts to make that feature available in the kernel are running into opposition. 

CPU designers will do everything they can to improve performance. With regard to memory accesses, "everything" can include caching operations, executing them out of order, combining multiple operations into one, and more. These optimizations do not affect a single CPU running in isolation, but they can cause memory operations to be visible to other CPUs in a surprising order. Unwary software running elsewhere in the system may see memory operations in an order different from what might be expected from reading the code; [this article](<https://lwn.net/Articles/576486/>) describes one simple scenario for how things can go wrong, and [this series on lockless algorithms](<https://lwn.net/Articles/844224/>) shows in detail some of the techniques that can be used to avoid problems related to memory ordering. 

The x86 architecture implements a model that is known as "total store ordering" (TSO), which guarantees that writes (stores) will be seen by all CPUs in the order they were executed. Reads, too, will not be reordered, but the ordering of reads and writes relative to each other is not guaranteed. Code written for a TSO architecture can, in many cases, omit the use of expensive barrier instructions that would otherwise be needed to force a specific ordering of operations. 

The Arm memory model, instead, is weaker, giving the CPU more freedom to move operations around. The benefits from this design are a simpler implementation and the possibility for better performance in situations where ordering guarantees are not needed (which is most of the time). The downsides are that concurrent code can require a bit more care to write correctly, and code written for a stricter memory model (such as TSO) will have (possibly subtle) bugs when run on an Arm CPU. 

The weaker Arm model is rarely a problem, but it seems there is one situation where problems arise: emulating an x86 processor. If an x86 emulator does not also emulate the TSO memory model, then concurrent code will likely fail, but emulating TSO, which requires inserting memory barriers, creates a significant performance penalty. It seems that there is one type of concurrent x86 code — games — that some users of Arm CPUs would like to be able to run; those users, strangely, dislike the prospect of facing the orc hordes in the absence of either performance or correctness. 

#### TSO on Arm

As it happens, some Arm CPU vendors understand this problem and have, as Hector Martin described in [this patch series](<https://lwn.net/ml/linux-kernel/20240411-tso-v1-0-754f11abfbff@marcan.st/>), implemented TSO memory models in their processors. Some NVIDIA and Fujitsu CPUs run with TSO at all times; Apple's CPUs provide it as an optional feature that can be enabled at run time. Martin's purpose is to make this capability visible to, and controllable by, user space. 

The series starts by adding a couple of new [`prctl()`](<https://man7.org/linux/man-pages/man2/prctl.2.html>) operations. `PR_GET_MEM_MODEL` will return the current memory model implemented by the CPU; that value can be either `PR_SET_MEM_MODEL_DEFAULT` or `PR_SET_MEM_MODEL_TSO`. The `PR_SET_MEM_MODEL` operation will attempt to enable the requested memory model, with the return code indicating whether it was successful; it is allowed to select a stricter memory model than requested. For the always-TSO CPUs, requesting TSO will obviously succeed. For Apple CPUs, requesting TSO will result in the proper CPU bits being set. Asking for TSO on a CPU that does not support it will, as expected, fail. 

Martin notes that the code is not new: ""This series has been brewing in the downstream Asahi Linux tree for a while now, and ships to thousands of users"". Interestingly, Zayd Qumsieh had posted [a similar patch set](<https://lwn.net/ml/linux-kernel/20240410211652.16640-1-zayd_qumsieh@apple.com/>) one day earlier, but that version only implemented the feature for Linux running in virtual machines on Apple CPUs. 

Unfortunately for people looking forward to faster games on Apple CPUs, neither patch set is popular with the maintainers of the Arm architecture code in the kernel. Will Deacon [expressed](<https://lwn.net/ml/linux-kernel/20240411132853.GA26481@willie-the-truck/>) his ""strong objection"", saying that this feature would result in a fragmentation of user-space code. Developers, he said, would just enable the TSO bit if it appears to make problems go away, resulting in code that will fail, possibly in subtle ways, on other Arm CPUs. Catalin Marinas, too, [indicated](<https://lwn.net/ml/linux-kernel/ZiKyWGKTw6Aqntod@arm.com/>) that he would block patches making this sort of implementation-defined feature available. 

Martin [responded](<https://lwn.net/ml/linux-kernel/28ab55b3-e699-4487-b332-f1f20a6b22a1@marcan.st/>) that fragmentation is unlikely to be a problem, and pointed to the different page sizes supported by some processors (including Apple's) as an example of how these incompatibilities can be dealt with. He said that, so far, nobody has tried to use the TSO feature for anything that is not an emulator, so abuse in other software seems unlikely. Keeping it out, he said, will not improve the situation: 

> There's a pragmatic argument here: since we need this, and it absolutely will continue to ship downstream if rejected, it doesn't make much difference for fragmentation risk does it? The vast majority of Linux-on-Mac users are likely to continue running downstream kernels for the foreseeable future anyway to get newer features and hardware support faster than they can be upstreamed. So not allowing this upstream doesn't really change the landscape vis-a-vis being able to abuse this or not, it just makes our life harder by forcing us to carry more patches forever. 

Deacon, though, [insisted](<https://lwn.net/ml/linux-kernel/20240419165826.GB4020@willie-the-truck/>) that, once a feature like this is merged, it will find uses in other software ""and we'll be stuck supporting it"". 

If this patch is not acceptable, it is time to think about alternatives. One is to, as Martin described, just keep it out-of-tree and ship it on the distributions that actually run on that hardware. A long history of addition by distributions can, at times, eventually ease a patch's way past reluctant maintainers. Another might be to just enable TSO unconditionally on Apple CPUs, but that comes with an overall performance penalty — about 9%, [according to Martin](<https://lwn.net/ml/linux-kernel/f6484dcd-ebf6-4b6f-be17-69b05539e33b@marcan.st/>). Another possibility was [mentioned](<https://lwn.net/ml/linux-kernel/87zftoqn7u.wl-maz@kernel.org/>) by Marc Zyngier, who suggested that virtual machines could be started with TSO enabled, making it available to applications running within while keeping the kernel out of the picture entirely. 

This seems like the kind of discussion that does not go away quickly. One of the many ways in which Linux has stood out over the years is in its ability to allow users to make full use of their hardware; refusing to support a useful hardware feature runs counter to that history. The concerns about potential abuse of this feature are also based in long experience, though. This is a case where the development community needs to repeat another part of its long history by finding a solution that makes the needed functionality available in a supportable way.
