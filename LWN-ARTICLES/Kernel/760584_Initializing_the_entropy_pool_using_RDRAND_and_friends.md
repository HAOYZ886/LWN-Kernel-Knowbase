---
title: Initializing the entropy pool using RDRAND and friends
url: https://lwn.net/Articles/760584/
date: "July 24, 2018"
category: "Random numbers; Security-Random number generation"
author: "By Jake Edge July 24, 2018"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
July 24, 2018

Random number generation in the kernel has garnered a lot of attention over the years. The tensions between the need for cryptographic-strength random numbers versus getting strong random numbers more quickly—along with the need to avoid regressions—has led to something of a patchwork of APIs. While it is widely agreed that waiting for a properly initialized random number generator (RNG) before producing random numbers is the proper course, opinions differ on what "properly" means exactly. Beyond that, waiting, especially early in the boot process, can be problematic as well. One solution would be to trust the RNG instructions provided by most modern processors, but that comes with worries of its own. 

Theodore Ts'o [posted a patch](<https://lwn.net/Articles/760121/>) on July 17 to add a kernel configuration option that would explicitly trust the CPU vendor's hardware RNG (e.g. the [`RDRAND` instruction](<https://en.wikipedia.org/wiki/RdRand>) for x86). Kernels built with `RANDOM_TRUST_CPU` will immediately initialize the random number pool using the architecture's facility, without waiting for enough entropy to accumulate from other sources; this means that the [`getrandom()` system call](<http://man7.org/linux/man-pages/man2/getrandom.2.html>) will not block. Waiting for systems to gather enough entropy [has been a problem in the past](<https://lwn.net/Articles/693189/>), especially for virtual machines and embedded systems where such entropy can be difficult to find. 

Currently, the kernel uses CPU-provided random number instructions as part of the process of mixing data into the entropy pool, without crediting any entropy to it; the patch would go further than that. Instead of waiting for enough entropy to be gathered at boot time, it would simply initialize the pool from the output of `RDRAND` (or other similar instructions). As Ts'o put it in the patch: ""I'm not sure Linux distro's will thank us for this. The problem is trusting the CPU [manufacturer] can be an emotional / political issue."" 

There are [some](<http://comments.gmane.org/gmane.comp.security.cryptography.randombit/4689>) who would rather these hardware RNG instructions not be used _at all_ for kernel random numbers. But blocking in `getrandom()` can lead to user space being unable to get random numbers as quickly as needed. By providing a configuration option, the kernel developers neatly avoid making a decision on whether to trust these hardware RNG instructions, but that leaves the decision to the distributions: 

Even if I were convinced that Intel hadn't backdoored RDRAND (or an NSA agent backdoored RDRAND for them) such that the NSA had a NOBUS (nobody but us) capability to crack RDRAND generated numbers, if we made a change to unconditionally trust RDRAND, then I didn't want the upstream kernel developers to have to answer the question, "why are you willing to trust Intel, but you aren't willing to trust a company owned and controlled by a PLA [People's Liberation Army] general?" (Or a company owned and controlled by one of Putin's Oligarchs, if that makes you feel better.) 

With this patch, we don't put ourselves in this position --- but we do put the Linux distro's in this position [instead]. The upside is it gives the choice to each person building their own Linux kernel to decide whether trusting RDRAND is worth it to avoid hangs due to userspace trying to get cryptographic-grade entropy early in the boot process. 

Sandy Harris [wondered](<https://lwn.net/Articles/760614/>) if there should be a value associated with the option, which would say how many bits of entropy the user thinks should be credited per 32-bit value from the hardware RNG. But Ts'o [pointed out](<https://lwn.net/Articles/760615/>) that his patch (which has a code diff roughly the same size as the text for its configuration option) does not mess with the entropy estimation. In any case, he is skeptical that kind of knob would truly be useful: 

In practice I doubt most people would be able to deal with a numerical dial, and I'm trying to [encourage] people to use getrandom(2). I view /dev/random as a legacy interface, and for most people a CRNG [Cryptographic-strength RNG] is quite sufficient. For those people who are super paranoid and want a "true random number generator" (and the meaning of that is hazy) because a CRNG is Not Enough, my recommendation these days is that they get something like an open hardware RNG solution, such as [ChaosKey](<https://altusmetrum.org/ChaosKey/>) from Altus Metrum. 

Other external hardware RNG devices were also mentioned as possibilities in the thread. 

In case distributors choose to enable this in their kernels, Yann Droneaud [would like to see](<https://lwn.net/Articles/760617/>) a kernel command-line option to disable it. That would likely make it easier for distributions to build kernels with it enabled since their users would have a way to override it without building their own kernel. Ts'o did not reply to that particular request directly, but his patch is meant as an RFC, so perhaps we will see that in the next version. In response to Droneaud, he did elaborate, including some thoughts on threat models: 

The trust model that we're using is this. The presumption is that (at least for US-based CPU [manufacturers]) the amount of effort needed to add a [blatant] backdoor to, say, the instruction scheduler and register management file is such that it couldn't be done by a single engineer, or even a very small set of engineers. Enough people would need to know about it, or would be able to figure out something untowards was happening, or it would be obvious through various regression tests, that it would be obvious if there was a generic back door in the CPU itself. This is a good thing, because ultimately we *have* to trust the general purpose CPU. If the CPU is actively conspiring against you, there really is no hope. 

However, the RDRAND unit is a small, self-contained thing, which is *documented* to use an AES whitener (e.g., it does an AES encryption as its last step). So presumably, a change to make the RDRAND unit effectively be: 

```
AES_ENCRYPT(NSA_KEY, COUNTER++)
```

Is much easier to hide or introduce. 

Laura Abbott was [set to test](<https://lwn.net/Articles/760621/>) the patch on a Fedora Rawhide kernel, but found that the hang at boot time waiting for random numbers had been fixed some other way. But she did think the option was a good idea: 

That said, I do think this is a beneficial option to have available because it actually fixes the underlying problem instead of hoping nobody else decides to block early in the bootup process. 

It is a simple change, though likely somewhat controversial—for distributions anyway. Another approach might be to simply change the kernel to always allow boot-time selection of whether to trust the CPU's hardware RNG. Defaulting that to "do not trust" would effectively leave things as they are today, but give users a way to decide for themselves—and take distributions off the hot seat if one truly exists.
