---
title: The future of GCC plugins in the kernel
url: https://lwn.net/Articles/851090/
date: "April 1, 2021"
category: "Build system-GCC plugins; GCC"
author: "April 1, 2021 This article was contributed by Marta Rybczyńska"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

April 1, 2021

This article was contributed by Marta Rybczyńska

The process of hardening the kernel can benefit in a number of ways from support by the compiler. In recent years, the [Kernel Self Protection Project](<https://lwn.net/Articles/698827/>) has brought this support from the [grsecurity](<https://grsecurity.net/>)/PaX patch set into the kernel in the form of [GCC plugins](<https://www.kernel.org/doc/html/latest/kbuild/gcc-plugins.html>); LWN looked into that process [back in 2017](<https://lwn.net/Articles/712161/>). A recent discussion has highlighted the fact that the use of GCC plugins brings disadvantages as well, and some developers would prefer to see those plugins replaced. 

The discussion started when Josh Poimboeuf [reported an issue](<https://lwn.net/ml/linux-kernel/efe6b039a544da8215d5e54aa7c4b6d1986fc2b0.1611607264.git.jpoimboe@redhat.com/>) he encountered when building out-of-tree modules with GCC plugins enabled. In his case, the compilation would fail when the GCC version used to compile the module was even slightly different from the one used to build the kernel. He included a patch to change the error he received into a warning and disable the affected plugin. Later in the thread, Justin Forbes [explained](<https://lwn.net/ml/linux-kernel/CAFbkSA0m1pqmXh29j6wJ9fG05yC72T1kNC0QU3rF7Oh2NoMwYQ@mail.gmail.com/>) how the problematic configuration came about; it happens within the Fedora continuous-integration system, which starts by building a current toolchain snapshot. Other jobs then compile out-of-tree modules with the new toolchain, without recompiling the kernel itself. Since GCC plugins were enabled, all jobs with out-of-tree modules have been failing.

The idea of changing the error into a warning was met with a [negative response](<https://lwn.net/ml/linux-kernel/CAK7LNAS=uOi=8xJU=NiKnXQW2iCazbErg_TX0gL9oayBiDffiA@mail.gmail.com/>) from the kernel build-system maintainer, Masahiro Yamada, who stated: ""We are based on the assumption that we use the same compiler for in-tree and out-of-tree"". Poimboeuf [responded](<https://lwn.net/ml/linux-kernel/20210125212755.jfwlqogpcarmxdgt@treble/>) that what he sees in real-world configurations doesn't match that assumption. Other kernel developers agreed with Yamada, though; Greg Kroah-Hartman [wrote](<https://lwn.net/ml/linux-kernel/YBAuddhlne9Dz/7M@kroah.com/>): 

> Have you not noticed include/linux/compiler.h and all of the different changes/workarounds we do for different versions of gcc/clang/intel compilers? We have never guaranteed that a kernel module would work that was built with a different compiler than the main kernel, and I doubt we can start now. 

In addition, Yamada [pointed out](<https://lwn.net/ml/linux-kernel/CAK7LNASOf=B_j5o=L=BL+vz_JK_d3QYUuHg8h+3SP6qOyYD86A@mail.gmail.com/>) that the use of the same compiler version for both the kernel and its modules has been accepted as an assumption in previous discussions. With clear disapproval from the kernel developers, the discussion seemed closed at that point.

#### The dislike for GCC plugins

It restarted, however, when Poimboeuf [came back](<https://lwn.net/ml/linux-kernel/20210302232649.y2tutffhxsblwqlb@treble/>) a few days later with another idea for solving his problem: recompiling all plugins when the GCC version changes. This was [refused](<https://lwn.net/ml/linux-kernel/CAK7LNAReuB5zUq_7S8ZG25+tdQowECDOK1rApYvkPCpHhPjK5w@mail.gmail.com/>) by Yamada, who noted that Ubuntu does not have the GCC mismatch problem, so the problem seemed to be specific to Fedora. Linus Torvalds also [disagreed](<https://lwn.net/ml/linux-kernel/CAHk-=wjR0CyaKU=6mXW9W+65L8h8DQuBdA2ZY2CfrPe6qurz3A@mail.gmail.com/>) with the proposal, but for another reason. For him there is no technical reason to recompile everything when the GCC version changes, but he expressed his concern on the usage and design of the GCC plugins in general. In a [followup message](<https://lwn.net/ml/linux-kernel/CAHk-=whA6zru0BaNm4uu5KyZe+aQpRScOnmc9hdOpO3W+xN9Xw@mail.gmail.com/>) he explained his reasoning in strong words: 

> The kernel gcc plugins _will_ go away eventually. They are an unmitigated disaster. They always have been. I'm sorry I ever merged that support. It's not only a maintenance nightmare, it's just a horrible thing and interface in the first place. It's literally BAD TECHNOLOGY. 

For Torvalds, the right way to implement such plugins is at the [intermediate representation](<https://en.wikipedia.org/wiki/Intermediate_representation>) (IR) level, but GCC plugins were designed differently for non-technical reasons (out of fear for non-free plugins, which LWN [covered](<https://lwn.net/Articles/301135/>) back in 2008). People who are interested in plugins should use Clang, as it has a clean IR and easily allows adding similar checks at the IR level, he said.

#### GCC plugins and their Clang equivalents

However, the removal of the kernel's GCC plugins does not seem likely in the near future. Kees Cook [commented on](<https://lwn.net/ml/linux-kernel/202103031334.8D898CA@keescook/>) the current status of the GCC plugins, their Clang equivalents, and why there is a user community for at least some of them. A number of the capabilities provided by the GCC plugins are not yet available with Clang — which many distributors are not using to build the kernel anyway. 

Currently the kernel supports the following plugins (located in [`scripts/gcc-plugins/`](<https://elixir.bootlin.com/linux/v5.11.10/source/scripts/gcc-plugins>)):

  * **cyc_complexity** computes the [cyclomatic complexity](<https://en.wikipedia.org/wiki/Cyclomatic_complexity>) of a function; it is one of the two initial example plugins, and likely has no users. 
  * **latent_entropy** adds entropy from the CPU execution. Cook sees no uses of it, especially since the addition of the the [jitter entropy](<https://lwn.net/Articles/642166/>) mechanism. There is no Clang support planned. 
  * The **per-task stack protector for arm32** provides stack protection for 32-bit ARM platforms; no Clang equivalent exists today even for 64-bit systems, Cook said. 
  * **randstruct** randomly changes the order of fields in kernel data structures that contain only function pointers, or are explicitly marked with `__randomize_layout`. There are two versions of this plugin: one complete and one restricted. The restricted version only changes the order of elements contained within the same cache line, which reduces the performance cost, but also the protection level. A Clang version [was submitted](<https://reviews.llvm.org/D59254>), but is stalled. Cook noted that security-conscious end users tend to enable this plugin, but distributors do not. 
  * **sancov** (which Cook didn't mention) helps fuzzing coverage by inserting a call to [`__sanitizer_cov_trace_pc()`](<https://elixir.bootlin.com/linux/v5.11.10/source/kernel/kcov.c#L189>) at the start of each basic block; it is used to determine which code blocks are being exercised. 
  * **stackleak** traces the kernel's stack depth so that it can overwrite the used stack with a pattern when returning to user space. There is no Clang support planned for now. 
  * **structleak** initializes structures that could be passed to user space. Clang has it implemented as the `-ftrivial-auto-var-init=zero` option; GCC is likely to gain support for that option as well at some point. 

The end result is that there is probably a reason to keep these plugins around for a while yet. 

Meanwhile, there were a couple of positive outcomes from the discussion. Along the way, it was realized that the plugins, which are highly sensitive to the GCC version they were built for, were not being rebuilt when that version changes. That had evidently been the case since the plugins were first added; that problem [was fixed](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/Makefile?h=v5.12-rc4&id=ce6ed1c4c9876c2880f52f18c41ef2a30d070bc5>) by Yamada, despite his rejection of this idea earlier in the discussion. As a solution for Poimboeuf's original problem, the developers [finally agreed](<https://lwn.net/ml/linux-kernel/CAK7LNARHoTnZ3gAvHgnYB4n-wYuboxC10A6zURh1ODGhxWd2yA@mail.gmail.com/>) to show a warning when there is a GCC version mismatch between the kernel and modules. It will be up to the user to decide if the difference is minor and safe, or if it is necessary to recompile the kernel.

The problem of the GCC version mismatch was not the only one noticed by Poimboeuf; he also [pointed out ](<https://lwn.net/ml/linux-kernel/20210125212755.jfwlqogpcarmxdgt@treble/>) the plugin build-system's dependency on the (optional) `gcc-plugin-devel` package. Even if the user has the same GCC version as used for the kernel compilation, but they do not have this package, plugins will be silently disabled, though the kernel compilation will succeed without any warning. This problem has not been addressed further.

#### Conclusions

The discussion covered a number of problems with the GCC plugins. It likely means that developers should be careful when enabling them. Poimboeuf's original problem got a solution of sorts in the form of a warning, which might start showing up in some systems. Users might be able to ignore the warning if the two GCC versions are close. When enabling plugins, developers should be careful to install `gcc-plugin-devel` first, otherwise their modules may be compiled in an unexpected way.

The future of GCC plugins in the kernel is not set in stone yet. Clang seems to be a preferred option for the hardening work, and this direction has been encouraged by Torvalds, but the existing GCC plugins (with one exception) do not have Clang equivalents. It seems that they will stay for at least some time.
