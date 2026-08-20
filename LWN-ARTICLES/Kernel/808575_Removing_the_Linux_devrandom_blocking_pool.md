---
title: Removing the Linux /dev/random blocking pool
url: https://lwn.net/Articles/808575/
date: "January 7, 2020"
category: Random numbers
author: "By Jake Edge January 7, 2020"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
January 7, 2020

The random-number generation facilities in the kernel have been reworked some over the past few months—but problems in that subsystem have been addressed over an [even longer time frame](<https://lwn.net/Kernel/Index/#Random_numbers>). The [most recent changes](<https://lwn.net/Articles/802360/>) were made to stop the `getrandom()` system call from blocking for long periods of time at system boot, but the underlying cause was the behavior of the blocking random pool. A recent patch set would remove that pool and it would seem to be headed for the mainline kernel. 

Andy Lutomirski [posted](<https://lwn.net/ml/linux-kernel/cover.1577088521.git.luto@kernel.org/>) version 3 of the patch set toward the end of December. It makes ""two major semantic changes to Linux's random APIs"". It adds a new `GRND_INSECURE` flag to the [`getrandom()`](<http://man7.org/linux/man-pages/man2/getrandom.2.html>) system call (though Lutomirski refers to it as `getentropy()`, which is implemented in glibc using `getrandom()` with fixed flags); that flag would cause the call to always return the amount of data requested, but with no guarantee that the data is random. The kernel would just make its best effort to give the best random data it has at that point in time. ""Calling it 'INSECURE' is probably the best we can do to discourage using this API for things that need security."" 

The patches also remove the blocking pool. The kernel currently maintains two pools of random data, one that corresponds to `/dev/random` and another for `/dev/urandom`, as described in this [2015 article](<https://lwn.net/Articles/658497/>). The blocking pool is the one for `/dev/random`; reads to that device will block (thus the name) until "enough" entropy has been gathered from the system to satisfy the request. Further reads from that file will also block if there is insufficient entropy in the pool. 

Removing the blocking pool means that reads from `/dev/random` behave like `getrandom()` with a `flags` value of zero (and turns the `GRND_RANDOM` flag into a noop). Once the cryptographic random-number generator (CRNG) has been initialized, reads from `/dev/random` and calls to `getrandom(..., 0)` will not block and will return the requested amount of random data. Lutomirski said: 

I believe that Linux's blocking pool has outlived its usefulness. Linux's CRNG generates output that is good enough to use even for key generation. The blocking pool is not stronger in any material way, and keeping it around requires a lot of infrastructure of dubious value. 

The changes were made with an eye toward ensuring that existing programs are not really affected; in fact, the problems with long waits for things like generating GnuPG keys will get better. 

This series should not break any existing programs. /dev/urandom is unchanged. /dev/random will still block just after booting, but it will block less than it used to. getentropy() with existing flags will return output that is, for practical purposes, just as strong as before. 

Lutomirski noted that there is still the open question of whether the kernel should provide so-called "true random numbers", which is, to a certain extent, what the blocking pool was meant to do. He can only see one reason to do so: ""compliance with government standards"". He suggested that if the kernel were to provide that, it should be done through an entirely different interface—or be punted to user space by providing a way for it to extract raw event samples that could be used to create such a blocking pool. 

Stephan Müller [suggested](<https://lwn.net/ml/linux-kernel/9872655.prSdhymlXK@positron.chronox.de/>) that his Linux random-number generator (LRNG) [patch set](<https://lwn.net/ml/linux-kernel/2722222.P16TYeLAVu%40positron.chronox.de/>) (now up to version 26) might be a way to provide true random numbers for applications that need them. The LRNG is ""fully compliant to [SP800-90B](<https://csrc.nist.gov/publications/detail/sp/800-90b/final>) requirements"", which makes it a solution to the governmental-standards problem. Matthew Garrett [objected](<https://lwn.net/ml/linux-kernel/20191226100334.bsh3b3dphs4j4cvx@srcf.ucam.org/>) to the term "true random data", noting that the devices being sampled could, in principle, be modeled accurately enough to make them predictable: ""We're not sampling quantum events here."" Müller [said](<https://lwn.net/ml/linux-kernel/8794771.pXx7ivobnm@tauon.chronox.de/>) that the term comes from the German AIS 31 standard to describe a random-number generator that only produces output ""at an equal rate as the underlying noise source produces entropy"". 

Beyond the terminology, though, having a blocking pool as is proposed by the LRNG patches will just lead to various problems, at least if it is available without privilege, Lutomirski [said](<https://lwn.net/ml/linux-kernel/888017FA-06A1-42EF-9FC0-46629138DA9E@amacapital.net/>): 

This doesn’t solve the problem. If two different users run stupid programs like gnupg, they will starve each other. 

As I see it, there are two major problems with /dev/random right now: it’s prone to DoS (i.e. starvation, malicious or otherwise), and, because no privilege is required, it’s prone to misuse. Gnupg is misuse, full stop. If we add a new unprivileged interface, gnupg and similar programs will use it, and we lose all over again. 

Müller [noted](<https://lwn.net/ml/linux-kernel/4820831.xlnk3tY4r2@tauon.chronox.de/>) that the addition of `getrandom()` will now allow GnuPG to use that interface since it will provide the needed guarantee that the pool has been initialized. From discussions with GnuPG maintainer Werner Koch, Müller believes that guarantee is the only reason GnuPG currently reads directly from `/dev/random`. But if there is an unprivileged interface that is subject to denial of service (like `/dev/random` today), it will be misused by some applications, Lutomirski [asserted](<https://lwn.net/ml/linux-kernel/E464D551-1631-4EA8-ADFB-852E16D29CA9@amacapital.net/>). 

Theodore Y. Ts'o, who is the maintainer of the Linux random-number subsystem, appears to have changed his mind along the way about the need for a blocking pool. He [said](<https://lwn.net/ml/linux-kernel/20191226140423.GB3158@mit.edu/>) that removing that pool would effectively get rid of the idea that Linux has a true random-number generator (TRNG), which ""is not insane; this is what the *BSD's have always done"". He, too, is [concerned](<https://lwn.net/ml/linux-kernel/20191227130436.GC70060@mit.edu/>) that providing a TRNG mechanism will just serve as an attractant for application developers. He also thinks that it is not really possible to guarantee a TRNG in the kernel, given all of the different types of hardware supported by Linux. Even making the facility only available to root will not solve the problem: 

Application programmers would give instructions requiring that their application be installed as root to be more secure, "because that way you can get access the _really_ good random numbers". 

Müller [asked](<https://lwn.net/ml/linux-kernel/15817620.rmTN4T87Wr@tauon.chronox.de/>) if Ts'o was giving up on the blocking pool implementation that he had added long ago. Ts'o [agreed](<https://lwn.net/ml/linux-kernel/20191227220857.GD70060@mit.edu/>) that he was; he is planning to take the patches from Lutomirski and is pretty strongly opposed to adding a blocking interface back into the kernel. 

The kernel can't offer up any guarantees about whether or not the noise source has been appropriately characterized. All say, a GPG or OpenSSL developer can do is get the vague sense that TRUERANDOM is "better" and of course, they want the best security, so of *course* they are going to try to use it. At which point it will block, and when some other clever user (maybe a distro release engineer) puts it into an init script, then systems will stop working and users will complain to Linus. 

For cryptographers and others who really _need_ a TRNG, Ts'o is also in favor of providing them a way to collect their own entropy in user space to use as they see fit. Entropy collection is not something that the kernel can reliably do on all of the different hardware that it supports, nor can it estimate the amount of entropy provided by the different sources, he said. 

The kernel shouldn't be mixing various noise sources together, and it certainly shouldn't be trying to claim that it knows how many bits of entropy that it gets when [it] is trying to play some jitter entropy game on a stupid-simple CPU architecture for IOT/Embedded user cases where everything is synchronized off of a single master oscillator, and there is no CPU instruction reordering or register renaming, etc., etc. 

You can talk about providing tools that try to make these estimations \--- but these sorts of things would have to be done on each user's hardware, and for most distro users, it's just not practical. 

So if it's just for cryptographers, then let it all be done in userspace, and let's not make it easy for GPG, OpenSSL, etc., to all say, "We want TrueRandom(tm); we won't settle for less". We can talk about how do we provide the interfaces so that those cryptographers can get the information they need so they can get access to the raw noise sources, separated out and named, and with possibly some way that the noise source can authenticate itself to the Cryptographer's userspace library/application. 

There was a bit of discussion about how that interface might look; there may be security implications for some of the events, for example. Ts'o [noted](<https://lwn.net/ml/linux-kernel/20191229144904.GB7177@mit.edu/>) that the keyboard scan codes (i.e. the keys pressed) are mixed into the pool as part of the entropy collection. ""Exposing this to userspace, even if it is via a privileged system call, would be... unwise."" It does seem possible that other event timings could provide some kind of side-channel information leak as well. 

So it would seem that a longtime feature of the Linux random-number subsystem is on its way out. Given the changes that the random-number subsystem have undergone recently, it effectively was only causing denial-of-service problems when it was used; there are now better ways to get the best random numbers that the kernel can provide. If a TRNG is still desired for Linux, that lack will need to be addressed in the future, but likely will not be done within the kernel itself.
