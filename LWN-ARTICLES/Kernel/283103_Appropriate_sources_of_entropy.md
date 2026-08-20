---
title: Appropriate sources of entropy
url: https://lwn.net/Articles/283103/
date: "May 21, 2008"
category: Random numbers; Security
author: "By Jake Edge May 21, 2008"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
May 21, 2008

A steady stream of random events allows the kernel to keep its entropy pool stocked up, which in turn allows processes to use the strongest random numbers that Linux can provide. Exactly which events qualify as random--and just how much randomness they provide--is sometimes difficult to decide. A recent move to eliminate a source of contributions to the entropy pool has worried some, especially in the embedded community. 

The kernel samples unpredictable events for use in generating random numbers, storing that data in the entropy pool. Entropy is a measure of the unpredictability or randomness of a data set, so the kernel estimates the amount of entropy each of those events contributes to the pool. Many kernels run on hardware that is lacking some of the traditional sources of entropy. In those cases, the timing of interrupts from network devices has been used as a source of entropy, but it has always been controversial, so it was recently proposed for removal. 

Two of the best sources of random data for the entropy pool--user interaction via a keyboard or mouse and disk interrupts--are often not present in embedded devices. In addition, some disk interfaces, notably ATA, do not add entropy, which extends the problem to many "headless" servers. But network interrupts are seen as a dubious source of entropy because they may be able to be observed, or manipulated, by an attacker. In addition, as network traffic rises, many network drivers turn off receive interrupts from the hardware, allowing the kernel to poll periodically for incoming packets. This would reduce entropy collection just at the time when it might be needed for encrypting the traffic. 

This is not the first time eliminating the `IRQF_SAMPLE_RANDOM` flag from network drivers has come up; we [looked at the issue](<http://lwn.net/Articles/182874/>) two years ago (though the flag was called `SA_SAMPLE_RANDOM` at that time). It has come up again, starting with a [query on linux-kernel](<https://lwn.net/Articles/283197/>) from Chris Peterson: ""Should network devices be allowed to contribute entropy to /dev/random?"" Jeff Garzik, kernel network device driver maintainer, [answered](<https://lwn.net/Articles/283303/>): ""I tend to push people to /not/ add IRQF_SAMPLE_RANDOM to new drivers, but I'm not interested in going on a pogrom with existing code."" 

For anyone that _is_ interested in such a pogrom, Peterson proposed a [patch to eliminate the flag](<http://lwn.net/Articles/282947/>) from the twelve network drivers that still use it. This sparked a long discussion on how to provide entropy for those devices that do not have anything else to use. While the actual contribution of entropy from network devices is questionable, mixing that data into the pool does not harm it, as long as no entropy credit--the current estimate of entropy in the pool--is awarded. Alan Cox [proposed a new flag to track sources like that](<https://lwn.net/Articles/283204/>): 

A more interesting alternative might be to mark things like network drivers with a new flag say IRQF_SAMPLE_DUBIOUS so that users can be given a switch to enable/disable their use depending upon the environment. 

Some were in favor of an approach like this, but Adrian Bunk [notes that](<https://lwn.net/Articles/283206/>): 

If he can live with dubious data he can simply use /dev/urandom . 

If a customer wants to use /dev/random and demands to get dubious data there if nothing better is available fulfilling his wish only moves the security bug from his crappy application to the Linux kernel. 

Part of the problem stems from a misconception about random numbers gotten from `/dev/random` versus those that are read from `/dev/urandom`, which we [described in a Security page article](<http://lwn.net/Articles/261804/>) last December. In general, applications should read from `/dev/urandom`. Only the most sensitive uses of random numbers--keys for GPG for example--need the entropy guarantee that `/dev/random` provides. In a system that is getting regular entropy updates, the quality of the random numbers from both sources is the same. 

There is still an initialization problem for some systems, though, as Ted Ts'o [points out](<https://lwn.net/Articles/283205/>): 

Hence, if you don't think the system hasn't run long enough to collect significant entropy, you need to distinguish between "has run long enough to collect entropy which is causes the entropy credits using a somewhat estimation system where we try to be conservative such that /dev/random will let you extract the number of bits you need", and "has run long enough to collect entropy which is unpredictable by an outside attacker such that host keys generated by /dev/urandom really are secure". 

A potential entropy source, even for embedded systems, is to sample other kernel and system parameters that are not predictable externally. Garzik [suggests](<https://lwn.net/Articles/283209/>): 

EGD demonstrates this, for example: <http://egd.sourceforge.net/> It looks at snmp, w, last, uptime, iostats, vmstats, etc. 

And there are plenty of untapped entropy sources even so, such as reading temperature sensors, fan speed sensors on variable-speed fans, etc. 

Heck, "smartctl -d ata -a /dev/FOO" produces output that could be hashed and added as entropy. 

Another source is from hardware random number generators. The kernel already has support for some, including the [VIA Padlock](<http://www.via.com.tw/en/initiatives/padlock/hardware.jsp>) that seems to be well thought of. Not all processors have such support, however. The [Trusted Platform Module](<http://en.wikipedia.org/wiki/Trusted_Platform_Module>) (TPM) does have random number generation and is becoming more widespread, especially in laptops, but there is no kernel `hw_random` driver for TPM. 

Garzik advocates adding a kernel driver for what he calls the "Treacherous Platform Module", but as others pointed out, it can all be done in user space using the [TrouSerS](<http://trousers.sourceforge.net/>) library. Even for the hardware random number generators that are supported in the kernel there is no automatic entropy collection, as it is left up to user space to decide whether to do that. This is done to try and keep policy decisions about the quality of the random data out of kernel code. 

Systems that wish to sample that data should use `rngd` to feed the kernel entropy pool. `rngd` will apply FIPS 140-2 tests to verify the randomness of the data before passing it to the kernel. Andi Kleen is [not in favor of that approach](<https://lwn.net/Articles/283212/>): 

Just think a little bit: system has no randomness source except the hardware RNG. you do your strange randomness verification. if it fails what do you do? You don't feed anything into your entropy pool and all your random output is predictable (just boot time) If you add anything predictable from another source it's still predictable, no difference. 

There is concern that some of the hardware random number generators are poorly implemented or could malfunction, so it would be dangerous to automatically add that data into the pool. Doing the FIPS testing in the kernel is not an option, leaving it up to user space applications to make the decision. There is nothing stopping any superuser process from adding bits to the entropy pool--no matter how weak--but the consensus is that the kernel itself must use sources it knows it can trust. 

Another instance of this problem--in a different guise--appears in a [discussion about random numbers for virtualized I/O](<https://lwn.net/Articles/283215/>), with Garzik asking: ""Has anyone yet written a "hw" RNG module for virt, that reads the host's random number pool?"" Rusty Russell responded with [a patch for a virtio "hardware" random number generator](<http://lwn.net/Articles/282721/>) as well as one that [adds it into his lguest hypervisor](<https://lwn.net/Articles/283217/>). The lguest patch reads data from the host's `/dev/urandom`, which is not where H. Peter Anvin [thinks it should come from](<https://lwn.net/Articles/283216/>): 

There is no point in feeding the host /dev/urandom to the guest (except for seeding, which can be handled through other means); it will do its own mixing anyway. The reason to provide anything at all from the host is to give it "golden" entropy bits. 

The virtio implementation only provides the `hw_random` implementation, thus it requires user space help to get entropy data into the kernel. Much like any process that can read `/dev/random`, lguest could exhaust the host entropy pool, so there was some discussion of limiting how much random data guests can request from the device. A guest implementation could then use a small pool of entropy read from the host to seed its own random number generator for the simulated hardware device. 

Removing the last remaining uses of `IRQF_SAMPLE_RANDOM` in network drivers seems likely, though some way to mix that data into the entropy pool without giving it any credit is still a possibility. With luck, that will encourage more effort into incorporating new sources of entropy using tools like EGD or, for systems that have it available, random number hardware. For systems that lack the traditional entropy sources, this should lead to a better initialized and fuller pool, while eliminating a potential attack by way of network packet manipulation.
