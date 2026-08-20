---
title: "Uniting the Linux random-number devices"
url: https://lwn.net/Articles/884875/
date: "February 16, 2022"
category: Random numbers
author: "By Jake Edge February 16, 2022"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
February 16, 2022

Blocking in the kernel's random-number generator (RNG)—causing a process to wait for "enough" entropy to generate strong random numbers—has always been controversial. It has also led to various kinds of problems over the years, from timeouts and delays caused by misuse in user-space programs to deadlocks and other problems in the boot process. That behavior has undergone a number of changes over the last few years and it looks possible that the last vestige of the difference between merely "good" and "cryptographic-strength" random numbers may go away in some upcoming kernel version. 

#### Random history

The history of the kernel RNG is long and somewhat twisty; there are two random-number devices in the kernel, `/dev/random` and `/dev/urandom`, that can be read in order to obtain the random data. `/dev/urandom` was always meant as the device for nearly everything to use, as it does not block; it simply provides the best random numbers that the kernel can provide at the time it is read. `/dev/random`, on the other hand, blocks whenever it does not have sufficient entropy to provide cryptographic-strength random numbers. That entropy comes from sources like interrupt timing for various kinds of devices (e.g. disk, keyboard, network) and hardware RNGs if they are available. `/dev/urandom` will log a warning message (once) if it is called before its pool is initialized (from the `random` pool once it has been initialized using gathered entropy), but it will provide output from its pseudorandom-number generator (PRNG) and never block. 

In 2014, for Linux 3.17, the [`getrandom()`](<https://man7.org/linux/man-pages/man2/getrandom.2.html>) system call was added [to provide a reliable way](<https://lwn.net/Articles/606141/>) for user-space applications to request random numbers even in the face of file-descriptor exhaustion or lack of access to the random devices (as might happen for an application running in a container). `getrandom()` was designed to use the `urandom` _pool_ , but only after it has been fully initialized from the `random` pool. So, while reads to `/dev/urandom` do not block, calls to `getrandom()` would until the requisite entropy is gathered. `getrandom()` callers can choose to use the `random` pool via a flag, which makes the call subject to the full entropy requirements for data coming from that pool. 

In 2019, an unrelated change to the ext4 filesystem [led to systems that would not boot](<https://lwn.net/Articles/800509/>) because it reduced the number of interrupts being generated, so the `urandom` pool did not get initialized and calls to `getrandom()` blocked. Since those calls were made early in the boot process, the system never came up to a point where enough entropy could be gathered, because the boot process was waiting for `getrandom()` to return—thus a deadlock resulted. The ext4 change was temporarily reverted for the 5.3 kernel and a [more permanent solution](<https://lwn.net/Articles/802360/>) was added by Linus Torvalds for 5.4. It used [CPU execution time jitter](<https://lwn.net/Articles/642166/>) as a source of entropy to ensure that the `random` pool initialized within a second or so. That technique is somewhat controversial, even Torvalds is somewhat skeptical of it, but it has been in place, and working as far as anyone can tell, for several years now. 

In 2020, the blocking nature of `/dev/random` was [changed](<https://lwn.net/Articles/808575/>) to behave like `getrandom()`, in that it would only block until it is initialized, once, and then would provide cryptographic-strength random numbers thereafter. Andy Lutomirski, who contributed the patches for that change, said: ""Linux's CRNG generates output that is good enough to use even for key generation. The blocking pool is not stronger in any material way, and keeping it around requires a lot of infrastructure of dubious value."" Those patches also added a `GRND_INSECURE` flag for `getrandom()` that would return "best effort" random numbers even if the pool was not yet initialized. 

As can be seen, the lines between the two devices have become rather blurrier over time. More of the history of the kernel RNG, going even further back in time, can be found in this [LWN kernel index entry](<https://lwn.net/Kernel/Index/#Random_numbers>). Given that the two devices have grown together, it is perhaps no surprise that a new proposal, to effectively eliminate the distinction, has been raised. 

#### No random blocking (for long)

Jason A. Donenfeld, who [stepped up as a co-maintainer](<https://lwn.net/ml/linux-kernel/20211130184315.258150-1-Jason@zx2c4.com/>) of the kernel's RNG subsystem a few months back, has been rather active in doing cleanups and making other changes to that code of late. On February 11, he [posted](<https://lwn.net/ml/linux-kernel/20220211210757.612595-1-Jason@zx2c4.com/>) an RFC—perhaps a ""request for grumbles"" in truth—patch proposing the removal of the ability for `/dev/urandom` to return data before the pool is initialized. It would mean that the kernel RNG subsystem would always block waiting to initialize, but always return cryptographic-strength random numbers thereafter (unless the `GRND_INSECURE` flag to `getrandom()` is used). Because of the changes made by Torvalds in 5.4, which Donenfeld calls the ""Linus Jitter Dance"", the maximum wait for initialization is minimal, so Donenfeld suggested the change: 

> So, given that the kernel has grown this mechanism for seeding itself from nothing, and that this procedure happens pretty fast, maybe there's no point any longer in having /dev/urandom give insecure bytes. In the past we didn't want the boot process to deadlock, which was understandable. But now, in the worst case, a second goes by, and the problem is resolved. It seems like maybe we're finally at a point when we can get rid of the infamous "urandom read hole". 

There are some potential hurdles to doing so, however. The jitter entropy technique relies on differences in timing when running the same code, which requires both a high-resolution CPU cycle counter and a CPU that appears to be nondeterministic (due to caching, instruction reordering, speculation, and so on). There are some architectures that do not provide that, however, so no entropy can be gathered that way. Donenfeld noted that non-Amiga m68k systems, two MIPS models (R6000 and R6000A), and, possibly, RISC-V would be affected; he wondered if there were other similarly affected architectures out there. He believes that the RISC-V code is not truly a problem, however, and no one has yet spoken up to dispute that. Meanwhile, setting those others aside might be the right approach: 

> If my general analysis is correct, are these ancient platforms really worth holding this back? I halfway expect to receive a few thrown tomatoes, an angry fist, and a "get off my lawn!", and if that's _all_ I hear, I'll take a hint and we can forget I ever proposed this. As mentioned, I do not intend to merge this unless there's broad consensus about it. But on the off chance that people feel differently, perhaps the Linus Jitter Dance is finally the solution to years of /dev/urandom kvetching. 

The proposed patch was fairly small; it simply eliminated the `file_operations` struct for `/dev/urandom` and reused the one for `/dev/random` in its place, thus making the two devices behave identically. It also shorted out the behavior of the `GRND_INSECURE` flag, but he later [said](<https://lwn.net/ml/linux-kernel/CAHmME9pkhqjLtwn2kt5SkSWBQ9UW+2WLGL9momZUmY3L2wDryQ@mail.gmail.com/>) that was something of a distraction. The main intent of his proposal was to do the following: 

> Right now, we have: 

```
>     /dev/random = getrandom(0)
>     /dev/urandom = getrandom(GRND_INSECURE)
>
```

> 
> This proposal is to make that: 

```
>     /dev/random = getrandom(0)
>     /dev/urandom = getrandom(0)
>
```

Torvalds had a [positive response](<https://lwn.net/ml/linux-kernel/CAHk-=wh+2jokbr4tpHA=ExebWKr=qp9RJ_uFrG2gYG4ChAjitg@mail.gmail.com/>) to the RFC. He said that the patch makes sense for architectures that have a cycle counter; the jitter entropy change has been active for two-and-a-half years without much complaint, so ""I think we can call that thing a success"". There may have been a few complaints about it, but: ""Honestly, I think all the complaints would have been from the theoretical posers that don't have any practical suggestions anyway"". Torvalds is known to have little patience for [theoretical concerns about cryptography](<https://lwn.net/Articles/828705/>) (or theoretical concerns about anything else, in truth). 

He did object to removing `GRND_INSECURE` for architectures that cannot do the jitter dance, since it is a way for user space to work around the lack of boot-time entropy, even if it is not at all secure: 

> Those systems are arguably broken from a randomness standpoint - what the h*ll are you supposed to do if there's nothing generating entropy \- but broken or not, I suspect they still exists. Those horrendous MIPS things were quite common in embedded networking (routers, access points - places that *should* care) 
> 
> [...] And almost nobody tests those broken platforms: even people who build new kernels for those embedded networking things probably end up using said kernels with an existing user space setup - where people have some existing saved source of pseudo-entropy. So they might not ever even trigger the "first boot problem" that tends to be the worst case. 

But, he said, he would be willing to apply the patch: ""at some point 'worry about broken platforms' ends up being too weak an excuse not to just apply it"". [According to Joshua Kinard](<https://lwn.net/ml/linux-kernel/fcab986b-d0bd-c798-de17-266abcdc7da2@gentoo.org/>), the two MIPS models in question were from the 1980s, not ever used in systems, and the kernel test for them in the random code ""was probably added as a mental exercise following a processor manual or such"". Maciej W. Rozycki [said](<https://lwn.net/ml/linux-kernel/alpine.DEB.2.21.2202122309560.34636@angie.orcam.me.uk/>) that there may have been a few systems using those models, but no Linux port was ever made for them. That might mean that the only problem systems are ""some m68k museum pieces"", Donenfeld [said](<https://lwn.net/ml/linux-kernel/CAHmME9ooEbgiv3DRk87ei+rUoVNMJthY7UuG_xCgm=kfMZAajw@mail.gmail.com/>). 

As Geert Uytterhoeven [pointed out](<https://lwn.net/ml/linux-kernel/CAMuHMdXDj+BVCs4Syg39vjnHGKcwJnCgDxrGoOiEFM_T4ojwPg@mail.gmail.com/>), though, the cycle-counter code for the Linux generic architecture, which is the default and starting point for new architectures, is hardwired to return zero. ""Several architectures do not implement get_cycles(), or implement it with a variant that's very similar or identical to the generic version."" David Laight [added](<https://lwn.net/ml/linux-kernel/63afd299c8df4495b27456a7f92aec75@AcuMS.aculab.com/>) a few examples (old x86, nios2) of architectures where that is the case. 

#### But what about my NetHack machine?

Lutomirski had a more prosaic [complaint](<https://lwn.net/ml/linux-kernel/fbdd43e1-a305-48d1-8ccb-2deffcb715f7@www.fastmail.com/>): 

> I dislike this patch for a reason that has nothing to do with security. Somewhere there’s a Linux machine that boots straight to Nethack in a glorious 50ms. If Nethack gets 256 bits of amazing entropy from /dev/urandom, then the machine’s owner has to play for real. If it repeats the same game on occasion, the owner can be disappointed or amused. If it gets a weak seed that can be brute forced, then the owner can have fun brute forcing it. 
> 
> If, on the other hand, it waits 750ms for enough jitter entropy to be perfect, it’s a complete fail. No one wants to wait 750ms to play Nethack. 

More seriously, he was concerned about devices like backup cameras or light bulbs that need to boot "immediately", and where the quality of the random numbers may not truly be a problem. The `GRND_INSECURE` escape hatch is there for just that reason. In a similar vein, Lennart Poettering was [worried](<https://lwn.net/ml/linux-kernel/YgoYnX97imub7KEB@gardel-login/>) that systemd would have to wait one second to get a seed for its hash tables, when it already has a mechanism to reseed the tables: 

> So, systemd uses (potentially half-initialized) /dev/urandom for seeding its hash tables. For that its kinda OK if the random values have low entropy initially, as we'll automatically reseed when too many hash collisions happen, and then use a newer (and thus hopefully better) seed, again acquired through /dev/urandom. i.e. if the seeds are initially not good enough to thwart hash collision attacks, once the hash table are actually attacked we'll replace the seeds with [something] better. For that all we need is that the random pool eventually gets better, that's all. 

It turns out that systemd is [already using `GRND_INSECURE`](<https://lwn.net/ml/linux-kernel/Ygps2W6MfxKkP5eD@gardel-login/>) on systems where it is available, so not changing that behavior, as was originally proposed, would neatly fix Poettering's concern. Donenfeld was completely amenable to [pulling the disabling of `GRND_INSECURE`](<https://lwn.net/ml/linux-kernel/YgpjfncV+C9FEZDc@zx2c4.com/>) from his patch; it is not really his primary focus with the proposal, as noted. 

Based on Torvalds's response, it would seem there are no huge barriers to removing the final distinction between `/dev/random` and `/dev/urandom`—other than the names, of course. If there are more architectures that cannot use the jitter technique, though, that distinction may live on, since Torvalds also thought there might be value in keeping ""the stupid stuff around as a 'doesn't hurt good platforms, might help broken ones'"". The code removal would not be huge, so it does not really provide much of a code simplification, Donenfeld said; it is more a matter of being able to eliminate the endless debate about which source of randomness to use on Linux. To that end, it seems like a worthwhile goal.
