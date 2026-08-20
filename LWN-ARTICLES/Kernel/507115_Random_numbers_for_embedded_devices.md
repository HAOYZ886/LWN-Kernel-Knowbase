---
title: Random numbers for embedded devices
url: https://lwn.net/Articles/507115/
date: "July 17, 2012"
category: "Random numbers; Security-Random number generation"
author: "By Jonathan Corbet July 17, 2012"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
July 17, 2012

Secure communications are dependent on good cryptography, and cryptography, in turn, is dependent on good random numbers. When cryptographic keys are generated from insufficiently-random values, they may turn out to be easily guessable by an attacker, leaving their user open to eavesdropping and man-in-the-middle attacks. For this reason, quite a bit of attention has been put into random number generation, but that does not mean that problems do not still exist. A set of patches intended for merging into the 3.6 kernel highlight some of the current concerns about random number generation in Linux. 

Computing systems traditionally do not have sources of true randomness built into them. So they have operated by attempting to extract randomness from the environment in which they operate. The fine differences in timing between a user's keystrokes are one source of randomness, for example. The kernel can also use factors like the current time, interrupt timing, and more. For a typical desktop system, such sources usually provide enough randomness for the system's needs. Randomness gets harder to come by on server systems without a user at the keyboard. But the hardest environment of all may be embedded systems and network routers; these systems may perform important security-related tasks (such as the generation of host keys) before any appreciable randomness has been received from the environment. 

As Zakir Durumeric, Nadia Heninger, J. Alex Halderman, and Eric Wustrow have [documented](<https://factorable.net/>), many of the latter class of systems are at risk, mostly as a result of keys generated with insufficient randomness and predictable initial conditions. They write: ""We found that 5.57% of TLS hosts and 9.60% of SSH hosts share public keys in an apparently vulnerable manner, due to either insufficient randomness during key generation or device default keys."" They were also able to calculate the actual keys used for a rather smaller (but still significant) percentage of hosts. Their site includes a key checker; concerned administrators may point it at their hosts to learn if their keys are vulnerable. 

Fixes for this problem almost certainly need to be applied at multiple levels, but kernel-level fixes seem particularly important since the kernel is the source for most random numbers used in cryptography. To that end, Ted Ts'o has put together [a set of patches](<https://lwn.net/Articles/507135/>) designed to improve the amount of randomness available in the system from when it first boots. Getting there involves making a number of changes. 

One of those is to fix the internal `add_interrupt_randomness()` function, which is used to derive randomness from interrupt timing. Use of this function has been declining in recent years, as a result of both its cost and concerns about the actual randomness of many interrupt sources. Ted's patch set tries to address the cost by batching interrupt-derived randomness on a per-CPU basis and only occasionally mixing it into the system-wide entropy pool. That mixing is also done with a new, lockless algorithm; this algorithm contains some small race conditions, but those could be seen to make the result even more random. An attempt is made to increase the amount of randomness obtained from interrupts by mixing in additional data, including the value of the instruction pointer at the time of the interrupt. After this change, adding randomness from interrupts should be fast and effective, so it is done by default for _all_ interrupts; the `IRQF_SAMPLE_RANDOM` interrupt flag no longer has any effect. 

Next, the patch set adds a new function: 

```
void add_device_randomness(const void *buf, unsigned int size);
```

The purpose is to allow drivers to mix in device-specific data that, while not necessarily random, is system-specific and unpredictable. Examples include serial, product, and manufacturer information from attached USB devices, the "write counter" from some realtime clock devices, and the MAC address from network devices. Most of this data should be random from the point of view of an attacker; it should help to prevent the situation where multiple, newly-booted devices generate the same keys. 

Finally, Ted's patch set also changes the use of the hardware random number generator built into a number of CPUs. Rather than return random numbers directly from the hardware, the code now mixes hardware random data into the kernel's entropy pool and generates random numbers from there. His reasoning is that using hardware random numbers directly requires placing a lot of trust in the manufacturer: 

It's unlikely that Intel (for example) was paid off by the US Government to do this, but it's impossible for them to prove otherwise --- especially since Bull Mountain is documented to use AES as a whitener. Hence, the output of an evil, trojan-horse version of RDRAND is statistically indistinguishable from an RDRAND implemented to the specifications claimed by Intel. Short of using a tunneling electronic microscope to reverse engineer an Ivy Bridge chip and disassembling and analyzing the CPU microcode, there's no way for us to tell for sure. 

Mixing hardware random data into the entropy pool helps to mitigate that threat. The first time this patch came around, Linus [rejected it](<https://lwn.net/Articles/507140/>), saying ""It would be a total PR disaster for Intel, so they have huge incentives to be trustworthy."" That opinion [was not universally shared](<https://lwn.net/Articles/506407/>), though, and the patch remains in the current set. Chances are it will be merged in its current form. 

An important part of the plan, though, is to get these patches into the stable updates despite their size. Then, with luck, device manufacturers will pick them up relatively quickly and stop shipping systems with a known weakness. Even better would be, as Ted [suggested](<https://lwn.net/Articles/507153/>), to make changes at the user-space levels as well. For example, delaying key generation long enough to let some randomness accumulate should improve the situation even more. But making things better at the kernel level is an important start.
