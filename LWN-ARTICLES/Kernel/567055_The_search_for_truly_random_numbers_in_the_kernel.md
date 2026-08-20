---
title: The search for truly random numbers in the kernel
url: https://lwn.net/Articles/567055/
date: "September 18, 2013"
category: "Random numbers; Security-Random number generation"
author: "By Jonathan Corbet September 18, 2013"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
September 18, 2013

The ongoing disclosures of governmental attempts to weaken communications security have caused a great deal of concern. Thus far, the evidence would seem to suggest that the core principles behind cryptography remain sound, and that properly encrypted communications can be secure. But the "properly encrypted" part is a place where many things can go wrong. One of those things is the generation of random numbers; without true randomness (or, at least, unpredictability), encryption algorithms can be far easier to break than their users might believe. For this reason, quite a bit of attention has been paid to the integrity of random number generation mechanisms, including the random number generator (RNG) in the kernel. 

Random number generation in Linux seems to have been fairly well thought out with no obvious mistakes. But that does not mean that all is perfect, or that improvements are not possible. The kernel's random number generator has been the subject of a few different conversations recently, some of which will be summarized here. 

#### Hardware random number generators

A program running on a computer is a deterministic state machine that cannot, on its own, generate truly random numbers. In the absence of a source of randomness from the outside world, the kernel is reduced to the use of a pseudo-random number generator (PRNG) algorithm that, in theory, will produce numbers that could be guessed by an attacker. In practice, guessing the results of the kernel's PRNG will not be an easy task, but those who concern themselves with these issues still believe that it is better to incorporate outside entropy (randomness) whenever it is possible. 

One obvious source of such randomness would be a random number generator built into the hardware. By sampling quantum noise, such hardware could create truly random data. So it is not surprising that some processors come with RNGs built in; the RDRAND instruction provided by some Intel processors is one example. The problem with hardware RNGs is that they are almost entirely impossible to audit; should some country's spy agency manage to compromise a hardware RNG, this tampering would be nearly impossible to detect. As a result, people who are concerned about randomness tend to look at the output of hardware RNGs with a certain level of distrust. 

Some [recently posted research [PDF]](<http://people.umass.edu/gbecker/BeckerChes13.pdf>) can only reinforce that distrust. The researchers (Georg T. Becker, Francesco Regazzoni, Christof Paar, and Wayne P. Burleson) have documented a way to corrupt a hardware RNG by changing the dopant polarity in just a few transistors on a chip. The resulting numbers still pass tests of randomness, and, more importantly, the hardware still looks the same at almost every level, regardless of whether one looks at the masks used or whether one looks at the chip directly with an electron microscope. This type of hardware compromise is thus nearly impossible to detect; it is also relatively easy to carry out. The clear conclusion is that hostile hardware is a real possibility; the corruption of a relatively simple and low-level component like an RNG is especially so. Thus, distrust of hardware RNGs would appear to be a healthy tendency. 

The kernel's use of data from hardware RNGs has been somewhat controversial from the beginning, with some developers wanting to avoid such sources of entropy altogether. The kernel's approach, though, is that using all available sources of entropy is a good thing, as long as it is properly done. In the case of a hardware RNG, the random data is carefully mixed into the buffer known as the "entropy pool" before being used to generate kernel-level random numbers. In theory, even if the data from the hardware RNG is entirely hostile, it cannot cause the state of the entropy pool to become known and, thus, it cannot cause the kernel's random numbers to be predictable. 

Given the importance of this mixing algorithm, it was a little surprising to see, earlier this month, [a patch](<https://lwn.net/Articles/567059/>) that would allow the user to request that the hardware RNG be used exclusively by the kernel. The argument for the patch was based on performance: depending entirely on RDRAND is faster than running the kernel's full mixing algorithm. But the RNG is rarely a performance bottleneck in the kernel, and the perceived risk of relying entirely on the hardware RNG was seen as being far too high. So the patch was not received warmly and had no real chance of being merged; sometimes it is simply better not to tempt users to compromise their security in the name of performance. 

H. Peter Anvin raised [a related question](<https://lwn.net/Articles/567064/>): what about hardware RNGs found in other components, and, in particular, in trusted platform module (TPM) chips? Some TPMs may have true RNGs in them; others are known to use a PRNG and, thus, are fully deterministic. What should the kernel's policy be with regard to these devices, which, for the most part, are ignored currently? The consensus seemed to be that no particular trust should be put into TPM RNGs, but that using some data from the TPM to seed the kernel's entropy pool at boot could be beneficial. Many systems have almost no entropy to offer at boot time, so even suspect random data from the TPM would be helpful early in the system's lifetime. 

#### Overestimated entropy

As noted above, the kernel attempts to pick up entropy from the outside world whenever possible. One source of entropy is the timing of device interrupts; that random data is obtained by (among other things) reading the time stamp counter (TSC) with a call to `get_cycles()` and using the least significant bits. In this way, each interrupt adds a little entropy to the pool. There is just one little problem, [pointed out](<https://lwn.net/Articles/567070/>) by Stephan Mueller: on a number of architectures, the TSC does not exist and `get_cycles()` returns zero. The amount of entropy found in a constant stream of zeroes is rather less than one might wish for; the natural consequence is that the kernel's entropy pool may contain less entropy than had been thought. 

The most heavily used architectures do not suffer from this problem; on the list of those that do, the most significant may be MIPS, which is used in a wide range of home network routers and other embedded products. As it turned out, Ted Ts'o had already [been working with the MIPS maintainers](<https://lwn.net/Articles/567074/>) to find a solution to this problem. He didn't like Stephan's proposed solution — reading a hardware clock if `get_cycles()` is not available — due to the expense; hardware clocks can take a surprisingly long time to read. Instead, he is hoping that each architecture can, somehow, provide some sort of rapidly increasing counter that can be used to contribute entropy to the pool. In the case of MIPS, there is a small counter that is incremented each clock cycle; it doesn't hold enough bits to work as a TSC, but it's sufficient for entropy generation. 

In the end, a full solution to this issue will take a while, but, Ted [said](<https://lwn.net/Articles/567075/>), that is not necessarily a problem: 

If we believed that /dev/random was actually returning numbers which are exploitable, because of this, I might agree with the "we must do SOMETHING" attitude. But I don't believe this to be the case. Also note that we're talking about embedded platforms, where upgrade cycles are measured in years --- if you're lucky. There are probably home routers still stuck on 2.6 [...] 

So, he said, it is better to take some time and solve the problem properly. 

Meanwhile, Peter came to another conclusion about the entropy pool: when the kernel writes to that pool, it doesn't account for the fact that it will be overwriting some of the entropy that already exists there. Thus, he said, the kernel's estimate for the amount of entropy in the pool is almost certainly too high. He put together [a patch set](<https://lwn.net/Articles/567076/>) to deal with this problem, but got little response. Perhaps that's because, as Ted [noted](<https://lwn.net/Articles/567077/>) in a different conversation, estimating the amount of entropy in the pool is a hard problem that cannot be solved without knowing a lot about where the incoming entropy comes from. 

The kernel tries to deal with this problem by being conservative in its accounting for entropy. Quite a few sources of unpredictable data are mixed into the pool with no entropy credit at all. So, with luck, the kernel will have a vague handle on the amount of entropy in the pool, and its mixing techniques and PRNG should help to make its random numbers thoroughly unpredictable. The end result should be that anybody wanting to attack the communications security of Linux users will not see poor random numbers as the easiest approach; in this world, one cannot do a whole lot better than that.
