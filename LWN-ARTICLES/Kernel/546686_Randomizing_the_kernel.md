---
title: Randomizing the kernel
url: https://lwn.net/Articles/546686/
date: "April 10, 2013"
category: Security
author: "By Jake Edge April 10, 2013"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
April 10, 2013

Address space layout randomization (ASLR) is a technique that makes it more difficult for security exploits to find the addresses they need to further their nefarious aims. [ASLR for user space](<https://lwn.net/Articles/121845/>) was [added in Linux 2.6.12](<https://lwn.net/Articles/126951/>) in 2005, but the kernel has always used fixed virtual addresses for its code. Back in 2011, Dan Rosenberg [proposed](<https://lwn.net/Articles/444503/>) changing to a random base address for x86 kernels; more recently, Kees Cook has [resurrected](<https://lwn.net/Articles/546035/>) and revised that patch set, with an eye toward getting it merged soon. 

If user-space code lives at known, fixed locations in the address space, exploits can use that information to reliably do their "job". If the kernel randomizes the location in memory where it loads the executable (along with the `mmap()` region and stack location), the attack code needs to determine the address some other way. One way would be to brute force the location of the function of interest. That can take time and provide some limited clues to an attentive administrator (lots of processes with segmentation faults, for example), but with enough patience, a local attack will succeed. Remote attacks are harder, but not impossible if the server is helpfully restarted after crashes caused by bad address guesses. 

For attacks against the kernel, things are a little different. If the kernel's layout is known, an attack can just use the address it needs, but if the kernel base address has been randomized, things are much trickier. A brute force attempt to find the address of interest is likely to result in a kernel oops, which may also lead to a panic depending on the nature of the error and the value of the "panic_on_oops" sysctl setting. Otherwise, some types of system monitoring might also pick up on repeated oops. 

For both the kernel and user space, though, information leaks of "interesting" addresses will allow attacks to be more (or completely) reliable. If the attacker (or exploit) can examine some leaked information to narrow down where the random addresses are, that will make attacks more effective. For user-space programs, most or all of the useful information leaks have been removed over the years. For kernel space, the process has been slower, but with settings like [`kptr_restrict`](<https://lwn.net/Articles/420403/>) and [`dmesg_restrict`](<https://lwn.net/Articles/414813/>) (both of which default off, but are recommended to be enabled in Cook's patches) many of the leaks have been plugged--but there are still more out there. 

For most kernels, though, finding addresses of interest is as easy as figuring out which distribution kernel is being run and then examining it (or its `System.map` file). That's where the idea of kernel ASLR (KASLR) comes from. If the kernel placed itself at a different, randomly chosen base address on each boot, exploits will have a harder time. KASLR is a bit of a misnomer, though, as the layout of the address space isn't randomized, just the base physical and virtual addresses where the kernel gets loaded. 

So, conceptually, Cook's patches are fairly easy to understand. They simply randomize the location of the kernel code each time it boots. The actual patches are a bit more difficult to follow. The first creates a new `aslr.S` x86 assembly code file that is used early in the boot to calculate a base address for the kernel. It uses the [`RDRAND` instruction](<http://en.wikipedia.org/wiki/RdRand>) available on recent Intel CPUs to get a random number from the hardware or the `RDTSC` instruction to create a pseudo-random number from the low-order bits in the time stamp counter (TSC). There are also some utility routines in that file. One calculates whether the kernel will fit at the physical address where it is to load, while another looks for "noaslr" on the kernel command line to disable KASLR. 

The second patch adds the ability to relocate the kernel for 64-bit systems. Currently, 32-bit x86 kernels can be made relocatable through the use of `arch/x86/tools/relocs`, but in order to support KASLR, `relocs` needed to work on 64-bit kernels too. Cook's patch refactors the source and changes the build to create `relocs_32` or `relocs_64` as appropriate. 

The heart of the changes comes with the third and final patch. It creates a `RANDOMIZE_BASE` build configuration parameter that must be enabled to get KASLR. The changes ensure that a random base address gets chosen at boot time, and that the kernel gets loaded there. The patch also makes the interrupt descriptor table (IDT) read-only for ""an added security benefit"". That particular piece is somewhat unrelated to the rest, which led Ingo Molnar to [request](<https://lwn.net/Articles/546713/>) that the read-only IDT piece be split out into a patch of its own. 

A recent [posting](<http://forums.grsecurity.net/viewtopic.php?f=7&t=3367>) by Brad Spengler and the PaX Team that was critical of KASLR was mentioned in the patch set, but Cook believes that ""the benefits of this feature in certain environments exceed the perceived weaknesses"". When asked for clarification, Cook [noted](<https://lwn.net/Articles/546715/>) that there are two main weaknesses cited: 

1- it depends on address location secrecy and leaks are common/easy.  
2- it has low entropy so attack success rates may be high. 

For certain environments, the address exposure issue is lessened, Cook continued. In particular, processes running on virtualized guests, remote processes, and contained processes (e.g. via [seccomp filters](<https://lwn.net/Articles/475043/>)) will all find information leaks difficult to access. The entropy for the existing patch is eight bits at best (it could be less depending on the [e820](<http://en.wikipedia.org/wiki/E820>) memory map). That gives attackers just a small (< 1%) chance to guess the right address. 

But, as H. Peter Anvin [pointed out](<https://lwn.net/Articles/546717/>), an attacker may not be targeting a single system: ""a bot might as well target 256 different systems and see what sticks"". Cook and others noted that the other 255 crashed systems should be helpful in alerting someone, whether the targeted systems were all run by the same organization or not. Cook also [elaborated](<https://lwn.net/Articles/546718/>) on the situation without KASLR: 

If an attacker targets multiple distinct systems across a wide range of system owners, instead of landing the exploit against all of them, they'll get less than 1% of them, and crash all the rest, removing them (for a while) from the target pool. Without KASLR, they would have landed 100% of the attacks. 

In addition, Molnar [noted](<https://lwn.net/Articles/546719/>) that attackers are often looking for quiet attacks that either succeed or fail without alerting anyone. ""Probabilistically crashing the kernel does not fit that requirement."" 

Though Anvin is a bit skeptical of KASLR (largely due to the small amount of entropy), most in the thread have been supportive of the idea. Cook mentioned that it has been running at Google and is being ported to Chrome OS, so it has some history behind it. The patches are not terribly intrusive, nor large (less than 1000 lines), though they do touch some potentially sensitive parts of the early boot code. Unless some stronger opposition materializes, it seems like something we could see in the mainline sometime this year.
