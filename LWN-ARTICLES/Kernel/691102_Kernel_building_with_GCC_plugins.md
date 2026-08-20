---
title: Kernel building with GCC plugins
url: https://lwn.net/Articles/691102/
date: "June 14, 2016"
category: "Build system-GCC plugins"
author: "By Jonathan Corbet June 14, 2016"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
June 14, 2016

It has long been understood that static-analysis tools can be useful in finding (and defending against) bugs and security problems in code. One of the best places to implement such tools is in the compiler itself, since much of the work required to analyze a program is already done in the compilation process. Despite the fact that GCC has had the ability to support security-oriented plugins for some years, the mainline kernel has never adopted any such plugins. That situation looks likely to change with the 4.8 kernel release, though. 

For many years, GCC famously did not support plugins out of a fear that proprietary plugins would undermine the free compiler. That roadblock ended in 2009, when the [GCC runtime library exemption](<https://lwn.net/Articles/301959/>) was rewritten. This library, which is needed by almost every program built with GCC, can be linked with proprietary code — but only if no non-GPLv3 plugins were used in the compilation process. The addition of that rule gave the powers that be at the Free Software Foundation the confidence that they could safely add a plugin mechanism to GCC. 

Relatively few plugins have materialized in any setting, perhaps because writing one requires a fairly deep understanding of how GCC works and the documentation available is not entirely helpful. (LWN ran [an introduction to creating GCC plugins](<https://lwn.net/Articles/457543/>) back in 2011). One group that did jump onto the plugin bandwagon, though, is [grsecurity](<https://grsecurity.net/>), where the ability to analyze — and transform — kernel code was quickly recognized as having a lot of potential. There were four plugins in the grsecurity patch set when LWN [took a look](<https://lwn.net/Articles/461696/>) in 2011. The current testing patch set from grsecurity shows twelve of them, performing a variety of functions: 

  * **Checker** incorporates some address-space checks normally performed separately with the [sparse](<https://lwn.net/Articles/689907/>) tool. 

  * **Colorize** simply adds color to some diagnostic output. 

  * **Constify** makes structures containing only function pointers `const`. 

  * **Initify** moves string constants that are only referenced in `__init` or `__exit` functions to the appropriate ELF sections. 

  * **Kallocstat** generates information on sizes passed to `kmalloc()`. 

  * **Kernexec** is there to ""to make KERNEXEC/amd64 almost as good as it is on i386""; it ensures that, for example, user-space pages are not executable by the kernel. 

  * **Latent_entropy** tries to generate entropy (randomness) from the kernel's execution; more on this one below. 

  * **Randomize_layout** reorganizes structure layout randomly. 

  * **Rap** implements grsecurity's "return address protection" mechanism, described in [this presentation [PDF]](<https://pax.grsecurity.net/docs/PaXTeam-H2HC15-RAP-RIP-ROP.pdf>). 

  * **Size_overflow** (described on [this page](<https://forums.grsecurity.net/viewtopic.php?f=7&t=3043>)) detects some integer overflows. 

  * **Stackleak** tracks kernel-stack usage so that the stack can be cleared on return to user space. 

  * **Structleak** forcibly clears structure fields if they might be copied to user space. 

These plugins have clear value to developers wishing to harden the kernel, and they are all free software (though many of them are GPLv2-only, meaning that they cannot be used to compile code needing the GCC runtime library; fortunately, the kernel does not use that library). So far, though, they remain unavailable to kernel developers and distributors, living only in the grsecurity patch set. There are no serious technical or legal obstacles keeping them out of the mainline, but nobody has made the effort to move them over — until now. 

#### Plugins go mainline

Recently, interest in hardening the mainline kernel has increased — or, perhaps more accurately, resistance to doing so has decreased. One obvious way of doing so is to try to bring some of the ideas found in grsecurity into the mainline kernel; that includes the plugin mechanism. To that end, the Linux Foundation's Core Infrastructure Initiative has funded Emese Révfy, the developer of some of the above-listed plugins, to bring this functionality into the mainline kernel. The resulting [patch set](<https://lwn.net/Articles/688491/>) has been through several rounds of review and is currently staged in linux-next for a probable 4.8 merge. 

Emese's patch set does not include all of the plugins listed above; indeed, it includes none of them. Instead, there are two relatively simple plugins provided as a sort of demonstration of how things can be done. One of them, called "sancov," inserts a tracing call at the beginning of each basic block of code. This feature is useful for anything requiring coverage tracking; it is aimed at the [syzkaller](<https://lwn.net/Articles/677764/>) fuzz tester in particular. 

The other included plugin calculates the "[cyclomatic complexity](<https://en.wikipedia.org/wiki/Cyclomatic_complexity>)" of each function in the kernel. This metric is a simple count of the number of possible paths through the function; a higher complexity count indicates more twisted code that, perhaps, is a more likely hiding place for bugs. Emese has [suggested](<https://lwn.net/Articles/691110/>) that it could be incorporated into the build-testing systems, where it could emit warnings when somebody adds a new function above a given complexity threshold. 

Your editor built an `allyesconfig` kernel with this plugin enabled; the result was nearly 620,000 complexity values printed to the output. According to this metric, the most complex function in the kernel, with a score of 817, is [`cache_alloc()`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/drivers/md/bcache/super.c#n1805>) in `drivers/md/bcache/super.c` — a demonstration of just how much complexity can be hidden in macros. Perhaps a more convincing demonstration is [`rt2800_init_registers()`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/drivers/net/wireless/ralink/rt2x00/rt2800lib.c#n4536>), a 450-line function weighing in at 586. The most complex core-kernel function is [`alloc_large_system_hash()`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/mm/page_alloc.c#n7074>), with a score of 278. 

The latent_entropy plugin from grsecurity [has been posted](<https://lwn.net/Articles/689145/>) as a separate patch set. This plugin tries to address the problem that systems often have very little entropy available immediately after boot. It adds code to initialization-time functions; each of those functions will generate a pseudo-random value when called and mix it into the entropy pool. That did not seem particularly random to a number of observers; the key, [according to "PaX Team",](<https://lwn.net/Articles/691118/>) is that the timing and sequencing of this mixing varies according to the interrupts raised during system boot. Ted Ts'o [commented](<https://lwn.net/Articles/691119/>) that this "entropy" might merely duplicate that obtained from the interrupt-timing measurements that are already done. He noted that mixing it in twice won't hurt, but it may not help much either. 

See [this 2012 message](<https://grsecurity.net/pipermail/grsecurity/2012-July/001093.html>) from PaX Team for more information on how the latent_entropy plugin works. 

As noted above, the plugin infrastructure and two simple plugins are currently poised to be merged for 4.8. The latent_entropy plugin is not in linux-next as of this writing, so it is likely to arrive later, if at all. But there is a whole set of existing plugins waiting for somebody to make the effort to bring them over and, even better, the potential for many other plugins to be written in the future. A pluggable compiler can be a potent tool for the checking and hardening of kernel code; the kernel community may have a lot to gain from making use of it.
