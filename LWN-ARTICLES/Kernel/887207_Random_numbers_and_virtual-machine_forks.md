---
title: "Random numbers and virtual-machine forks"
url: https://lwn.net/Articles/887207/
date: "March 11, 2022"
category: "Random numbers; Security-Random number generation; Virtualization"
author: "By Jonathan Corbet March 11, 2022"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
March 11, 2022

One of the key characteristics of a random-number generator (RNG) is its unpredictability; by definition, it should not be possible to know what the next number to be produced will be. System security depends on this unpredictability at many levels. An attacker who knows an RNG's future output may be able to eavesdrop on (or interfere with) network conversations, compromise cryptographic keys, and more. So it is a bit disconcerting to know that there is a common event that can cause RNG predictability: the forking or duplication of a virtual machine. Linux RNG maintainer Jason Donenfeld is working on a solution to this problem. 

The kernel's RNG maintains an "entropy pool" from which random numbers are derived. As randomness from the environment is harvested, it is mixed into the pool, keeping the level of entropy up. Every running system has its own pool, naturally, with its own internal state. If two systems were to somehow end up with their entropy pools containing the same data, they would produce the same sequence of random numbers, for a while at least. That is something that should never happen. 

But, as Donenfeld pointed out in [a patch set](<https://lwn.net/ml/linux-kernel/20220223131231.403386-1-Jason@zx2c4.com/>) first released in February, there is a way that two systems can end up with the same entropy-pool content. If a running virtual machine is somehow duplicated, the entropy pool will be duplicated with it. This can happen if a machine is checkpointed and restored, or if it forks for any reason. Once the restored or forked machine starts running, it will reproduce the sequence of random data created by the previous instance until the addition of new entropy perturbs the pool. 

Microsoft, it seems, has already addressed this concern in Windows; the solution takes the form of a "virtual-machine ID" made available via the ACPI firmware. When a virtual machine forks or is restarted, the ID is changed. The kernel can watch this value and, on noticing that it has changed, conclude that some sort of virtual-machine fork has occurred and that action is necessary to keep the random-data stream unique. Some virtualization systems, including QEMU, have implemented this functionality, so it makes sense for Linux to make use of it as well. 

The patch set thus adds a new "vmgenid" driver that makes the virtual-machine ID available to the kernel. When this driver is notified (by the ACPI firmware) of a change, it checks the ID and, if that has changed, calls a new function (`crng_vm_fork_inject()`) to inform the RNG that it needs to muddy up the entropy pool. This is done by mixing in that same virtual-machine ID. It is not claimed to be the ultimate in security, but it does address the immediate problem; Donenfeld intends to merge this work for 5.18. 

This project does not stop there, though; in a later email, Donenfeld [described](<https://lwn.net/ml/linux-kernel/Yh4+9+UpanJWAIyZ@zx2c4.com/>) the changes he would like to make next. He started by complaining about the design of Microsoft's solution, which has a race condition designed into it. The kernel cannot respond to a virtual-machine fork until it notices that the generation ID has changed; the new virtual machine could run for some time before that happens, and it will generate duplicate random numbers during that time. It would have been better, he said, to provide a simple "generation counter" that could be quickly polled by the CPU every time random data is requested; that would allow a change to be caught immediately. ""But that's not what we have, because Microsoft didn't collaborate with anybody on this, and now it's implemented in several hypervisors"". 

Having gotten that off his chest, he proceeded to the real task at hand: propagating the news about a virtual-machine fork to other interested kernel subsystems. He originally envisioned creating a new power-management event to serve as a sort of notifier, but that was seen as an awkward fit; a virtual-machine fork isn't really related to power management in the end. So Donenfeld [posted a new solution](<https://lwn.net/ml/linux-kernel/20220301231038.530897-1-Jason@zx2c4.com/>) creating a separate notifier (using the kernel's existing notifier mechanism) to inform subsystems. The [first user](<https://lwn.net/ml/linux-kernel/20220301231038.530897-4-Jason@zx2c4.com/>) is, unsurprisingly, the WireGuard VPN, which needs to know about such events: 

> When a virtual machine forks, it's important that WireGuard clear existing sessions so that different plaintext is not transmitted using the same key+nonce, which can result in catastrophic cryptographic failure. 

User-space code may benefit from knowing about virtual-machine forks as well; for example, the Samba server may want to reset sessions in that situation. For user space, Donenfeld [proposes](<https://lwn.net/ml/linux-kernel/20220309215907.77526-1-Jason@zx2c4.com/>) adding a new virtual file that programs can poll for VM-fork notifications. This file is currently located in `/proc/sys`, even though it is not a true sysctl knob in that it cannot be written to as a way of tuning system behavior. 

Response to this work has been positive, overall; kernel developers tend to have little appetite for catastrophic cryptographic failure. That said, Greg Kroah-Hartman did [observe](<https://lwn.net/ml/linux-kernel/Yh8wjrf7HVf56Anw@kroah.com/>): ""It seems crazy that the 'we just were spawned as a new vm' notifier is based in the random driver, but sure, put it here for now!"" So this work, too, seems destined for merging for the 5.18 kernel release. That should help to close a vulnerability that many of us may not have ever been aware existed.
