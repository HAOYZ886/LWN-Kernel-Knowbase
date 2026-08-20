---
title: "Maintaining the kernel's web of trust"
url: https://lwn.net/Articles/798230/
date: "September 4, 2019"
category: "Security-Patch verification"
author: "By Jonathan Corbet September 4, 2019"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
September 4, 2019

A typical kernel development cycle involves pulling patches from over 100 repositories into the mainline. Any of those pulls could conceivably bring with it malicious code, leaving the kernel (and its users) open to compromise. The kernel's web of trust helps maintainers to ensure that pull requests are legitimate, but that web has become difficult to maintain in the wake of the recent [attacks on key servers](<https://lwn.net/Articles/792366/>) and other problems. So now the kernel community is taking management of its web of trust into its own hands. 

#### Some history

As recently as 2011, there was no mechanism in place to verify the provenance of pull requests sent to kernel maintainers. If an emailed request looked legitimate, and the proposed code changes appeared to make sense, then the requested pull would generally be performed. That degree of openness makes for a low-friction development experience, but it also leaves the project open to at least a couple types of attacks. Email is easy to forge; an attacker could easily create an email that appeared to be from a known maintainer, but which requested a pull from a malicious repository. 

The risk grows greater if an attacker somehow finds a way to modify a maintainer's repository (on kernel.org or elsewhere); then the malicious code would be coming from a trusted location. The chances of a forged pull request from a legitimate (but compromised) repository being acted on are discouragingly high. 

The [compromise of kernel.org](<https://lwn.net/Articles/464233/>) in 2011 focused minds on this problem. By all accounts, the attackers had no idea of the importance of the machine they had taken over, so they did not even try to tamper with any of the repositories kept there. But they _could_ have done such a thing. Git can help developers detect and recover from such attacks, but only to an extent. What the community really needs is a way to know that a specific branch or tag proposed for pulling was actually created by the maintainer for the relevant subsystem. 

One action that was taken was to transform kernel.org from a machine managed by a small number of kernel developers in their spare time into a carefully thought-out system run by full-time administrators supported by the Linux Foundation. The provision of shell accounts to hundreds of kernel developers was belatedly understood to be something other than the best of ideas, so that is no longer done. No system is immune, but kernel.org has become a much harder target than before, so repositories stored there should be relatively safe. 

The other thing that was done, though, was the establishment of a web of trust based on public-key encryption with GnuPG. When a subsystem maintainer readies a branch for pushing to a higher-level maintainer, they should apply a signed tag to the topmost commit. The receiving maintainer can then verify the signature and be sure that the series of commits they are pulling is what the maintainer had in mind. As can be seen from [this article](<https://lwn.net/Articles/737093/>), not all maintainers are using signed tags, but their use has been growing over time. Adoption has been slowed a bit because Linus Torvalds does not require signed tags for pulls from kernel.org repositories. 

A signed tag by itself does not mean much; the other half of the problem is that the pulling maintainer must be able to verify that the key used to sign that tag actually belongs to the developer it claims to. That is where the web of trust comes in. If Torvalds can be convinced that a given key belongs to a specific subsystem maintainer, he can sign that key; other maintainers can then trust that the key is as advertised (as long as they trust Torvalds, anyway). In the kernel community, all roads lead to Torvalds, but he does not have to personally sign every maintainer's key; as long as there is a path of trusted signatures leading to him, a key will be trusted within the community. 

The kernel's web of trust was bootstrapped in a painful key-signing session at the 2011 Kernel Summit; thereafter, new developers have had to convince others to sign their keys at conferences or other gatherings. Until recently, PGP key servers were used to hold keys and any signatures attached to them. A given maintainer's key could be easily fetched and, if the signature chain checked out, trusted. The attacks on the signature mechanism, including the attachment of thousands of bogus signatures to public keys, have taken the key servers out of the picture, though, leaving the community without a way to maintain its web of trust. 

#### pgpkeys.git

Konstantin Ryabitsev, the lead administrator for kernel.org, has [stepped into this void](<https://lwn.net/ml/linux-kernel/20190830143027.cffqda2vzggrtiko@chatter.i7.local/>). After investigating a number of key-server alternatives, he concluded that none of them were fit for the purpose; the code is unmaintained and there is little interest in the development of web-of-trust systems in general at this point. So the alternatives are to give up on the web of trust as well or to come up with a new solution. Dropping the web of trust is not an appealing option: 

Unfortunately, if we abandon the web of trust completely, we will have to go back to relying on kernel.org infrastructure as the source of trust. Kernel.org has been hacked in the past -- ever since then our goal has always been to keep developers as the sole and only source of truth. This requirement is why we cannot and should not abandon the developer web of trust and must keep it going, at least in parallel to the [[web key directory](<https://wiki.gnupg.org/WKD>)] and similar efforts. 

So Ryabitsev has created [a new Git repository](<https://git.kernel.org/pub/scm/docs/kernel/pgpkeys.git>) to hold keys for kernel developers. It has been populated with keys used in pull requests in the past, along with the signatures on those keys. But, to avoid signature attacks, only signatures made with other keys stored in the repository are retained; that is sufficient to build the web of trust while eliminating the results of any signature spamming that might have taken place. There is also a set of SVG files showing how each key fits into the web of trust; to take a random example: 

> ![\[web-of-trust graph\]](https://static.lwn.net/images/2019/borkman-key.svg)

There is also, naturally, a way to make updates to this repository. Keys can be sent to an email address; after verification, they will be added to (or updated in) the repository. Finally, there is a script that can be used to automatically load all of the keys in this repository into one's personal GnuPG keyring. By running this script periodically, any developer can keep a copy of the entire kernel web of trust at hand. 

To some, this may seem like a rearguard action aimed at propping up the web-of-trust concept when that idea is generally falling out of favor. It may well be true that the web of trust, as originally conceived with PGP many years ago, cannot scale to the Internet as a whole. It can still work, though, for relatively small communities, as the Debian project (for example) has shown for years. A Git repository full of keys will not solve the world's authentication problems, but it may well prove sufficient for the task of keeping kernel pull requests secure.
