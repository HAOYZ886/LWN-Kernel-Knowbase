---
title: "Why RISC-V doesn't (yet) support KVM"
url: https://lwn.net/Articles/856685/
date: "May 20, 2021"
category: "Architectures-RISC-V; KVM; Virtualization-KVM"
author: "By Jonathan Corbet May 20, 2021"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
May 20, 2021

The [RISC-V](<https://riscv.org/>) CPU architecture has been gaining prominence for some years; its relatively open nature makes it an attractive platform on which a number of companies have built products. Linux supports RISC-V well, but there is one gaping hole: there is no support for virtualization with [KVM](<https://en.wikipedia.org/wiki/Kernel-based_Virtual_Machine>), despite the fact that a high-quality implementation exists. A recent attempt to add that support is shining some light on a part of the ecosystem that, it seems, does not work quite as well as one would like. 

Linux supports a number of virtualization mechanisms, but KVM is generally seen as the native solution. It provides a standard interface across systems, but much of KVM is necessarily architecture-specific, since the mechanisms for supporting virtualization vary from one processor to the next. Thus, architectures that support KVM generally have a `kvm` directory nestled in with the rest of the architecture-specific code. 

Given that, some eyebrows were raised when Anup Patel's [patch series adding RISC-V KVM support](<https://lwn.net/ml/linux-kernel/20210519033553.1110536-1-anup.patel@wdc.com/>) deposited the architecture-specific code into the staging directory instead. Staging is normally used for device drivers that do not meet the kernel's standards for code quality; if all goes well they are improved and eventually "graduate" out of the staging directory. It is not usually a place for architecture support. So staging maintainer Greg Kroah-Hartman was [quick to ask](<https://lwn.net/ml/linux-kernel/YKSa48cejI1Lax+%2F@kroah.com/>) why things were being done that way. 

The answer comes down to the patch-acceptance policy for RISC-V code, found in [Documentation/riscv/patch-acceptance.rst](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/riscv/patch-acceptance.rst>), which reads: 

> We'll only accept patches for new modules or extensions if the specifications for those modules or extensions are listed as being "Frozen" or "Ratified" by the RISC-V Foundation. (Developers may, of course, maintain their own Linux kernel trees that contain code for any draft extensions that they wish.) 

Virtualization for RISC-V is described by the [hypervisor extension](<https://github.com/riscv/riscv-isa-manual/blob/master/src/hypervisor.tex>) specification. As Patel [explained](<https://lwn.net/ml/linux-kernel/CAAhSdy18qySXbUdrEsUe-KtbtuEoYrys0TcmsV2UkEA2=7UQzw@mail.gmail.com/>), that extension has not yet been approved: 

> The KVM RISC-V patches have been sitting on the lists for almost 2 years now. The requirements for freezing RISC-V H-extension (hypervisor extension) keeps changing and we are not clear when it will be frozen. In fact, quite a few people have already implemented RISC-V H-extension in hardware as well and KVM RISC-V works on real HW as well. 
> 
> Rationale of moving KVM RISC-V to drivers/staging is to continue KVM RISC-V development without breaking the Linux RISC-V patch acceptance policy until RISC-V H-extension is frozen. 

It is fair to say that Kroah-Hartman [was not impressed](<https://lwn.net/ml/linux-kernel/YKSgcn5gxE%2F4u2bT@kroah.com/>); he stated that circumventing policies in other parts of the kernel tree is not the purpose of the staging directory, so the RISC-V KVM code would not be accepted there. Paolo Bonzini, the overall maintainer for KVM, [added](<https://lwn.net/ml/linux-kernel/d7d5ad76-aec3-3297-0fac-a9da9b0c3663@redhat.com/>) that ""the RISC-V acceptance policy as is just doesn't work"", as demonstrated by the fact that developers are having to work around it. This is especially unfortunate here, he said, because the RISC-V KVM implementation ""is also a very good example of how to do things right"". He [went on](<https://lwn.net/ml/linux-kernel/daa30135-8757-8d33-a92e-8db4207168ff@redhat.com/>) to suggest that there may be some players out there who benefit from the slowing down of patches on their way into the kernel. 

Kroah-Hartman [responded](<https://lwn.net/ml/linux-kernel/YKUZbb6OK+UYAq+t@kroah.com/>) that slowing down the merging of useful code is ""horrible"" and that the job of the kernel community is to make hardware work; a policy that prevents the merging of good code that adds support for existing hardware does not make sense. He asked the RISC-V maintainers to explain this policy; they have yet to answer that question. Back in April, though, RISC-V maintainer Palmer Dabbelt [acknowledged](<https://lwn.net/ml/linux-kernel/mhng-d64da1be-bacd-4885-aaf2-fea3c763418c@palmerdabbelt-glaptop/>) that the acceptance policy is not producing the desired results: 

> My goal with the RISC-V stuff has always been getting us to a place where we have real shipping products running a software stack that is as close as possible to the upstream codebases. I see that as the only way to get the software stack to a point where it can be sustainably maintained. The "only frozen extensions" policy was meant to help this by steering vendors towards a common base we could support, but in practice it's just not working out. 

He added that the policy could change, but only when there is a new policy for patch acceptance in place that everybody agrees to. Co-maintainer Paul Walmsley, though, [placed the blame](<https://lwn.net/ml/linux-kernel/5b988c4e-25e9-f2b9-b08d-35bc37a245e4@sifive.com/>) with RISC-V International's specification process and said that any fixes should be applied there instead. 

It is not that hard to understand why the RISC-V maintainers might not want to attempt to support every nonstandard CPU that is out there. The open nature of RISC-V makes it relatively easy for anybody to create their own variant, and supporting them all could become unworkable. The counterpoint is, as Kroah-Hartman said, that the kernel's job is to run on the hardware that is out there. Blocking support for shipping systems can only have the effect of pushing those systems toward vendor-supplied kernels with a lot of out-of-tree code — an unfortunate outcome for what is supposed to be an open architecture. 

When the subject is support for a feature as fundamental as virtualization, the question becomes even more urgent. Hopefully this episode will lead to a rethinking of the patch-acceptance policies for the RISC-V architecture. Failing that, Kroah-Hartman has [signaled his willingness](<https://lwn.net/ml/linux-kernel/YKTsyyVYsHVMQC+G@kroah.com/>) to allow that support into staging if there is truly no alternative. So Linux seems likely to gain KVM support for RISC-V in the relatively near future, even if it's necessary to circumvent the architecture's maintainers to do it.
