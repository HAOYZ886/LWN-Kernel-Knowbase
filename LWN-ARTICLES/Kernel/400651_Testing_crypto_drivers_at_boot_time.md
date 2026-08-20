---
title: Testing crypto drivers at boot time
url: https://lwn.net/Articles/400651/
date: "August 18, 2010"
category: Build system
author: "By Jake Edge August 18, 2010"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
August 18, 2010

Developers, understandably, want their code to be used, but turning new features on by default is often thought to be taking things a bit too far. Herbert Xu and other kernel crypto subsystem developers recently ran afoul of this policy when a new option controlling the self-testing the crypto drivers at boot time was set to "yes" by default. They undoubtedly thought that this feature was important--bad cryptography can lead to system or data corruption--but Linux has a longstanding policy that features should default to "off". When David Howells ran into a [problem](<https://lwn.net/Articles/400704/>) caused by a bug when loading the cryptomgr module, Linus Torvalds was quick to sharply remind Xu of that policy. 

The proximate cause of Howells's problem was that the cryptomgr was returning a value that made it appear as if it was not loaded. That caused a cascade of problems early in the boot sequence when the module loader was trying to write an error message to `/dev/console`, which had not yet been initialized. Xu sent out a [patch](<https://lwn.net/Articles/400720/>) to fix that problem, but Howells's bisection pointed to a commit that added a way to disable boot-time crypto self-tests--defaulted to running the tests. 

Torvalds was characteristically [blunt](<https://lwn.net/Articles/400721/>): ""People always think that their magical code is so important. I tell you up-front that [it] absolutely is not. Just remove the crap entirely, please."" He was unhappy that, at least by default, everyone would be running these self-tests every time they boot. But Xu was [worried](<https://lwn.net/Articles/400730/>) about data corruption and potentially flaky crypto hardware: 

The purpose of these tests are to make a particular driver or implementation available only if it passes them. So your encrypted disk/file system will not be subject to a hardware/software combo without at least some semblance of testing. 

The last thing you want to is to upgrade your kernel with a new hardware crypto driver that detects that you have a piece of rarely- used crypto [hardware], decides to use it and ends up making your data toast. 

But Torvalds was [unconvinced](<https://lwn.net/Articles/400734/>): ""The _developer_ had better test the thing. That is absolutely _zero_ excuse for then forcing every boot for every poor user to re-do the test over and over again."". Others were not so sure, however. Kyle Moffett [noted](<https://lwn.net/Articles/400735/>) that he had been personally bitten by new hardware crypto drivers that failed the self-tests--thus falling back to the software implementation--so he would like to see more testing: 

So there are unique and compelling reasons for default-enabled basic smoke tests of cryptographic support during boot. To be honest, the test and integration engineer in me would like it if there were more intensive in-kernel POST tests that could be enabled by a kernel parameter or something for high-reliability embedded devices. 

Basically Torvalds's point was that making every user pay the cost to run the self-tests at boot time was too high. The drivers _should_ be reliable or they shouldn't be in the kernel. He [continued](<https://lwn.net/Articles/400736/>): ""And if you worry about alpha-particles, you should run a RAM test on every boot. But don't ask _me_ to run one."" 

Though Xu posted a [patch](<https://lwn.net/Articles/400737/>) to default the self-tests to "off", it has not yet made its way into the mainline. Given Torvalds's statements, though, that will probably happen relatively soon. If distributions disagree with his assessment, they can, and presumably will, enable the tests for their kernels.
