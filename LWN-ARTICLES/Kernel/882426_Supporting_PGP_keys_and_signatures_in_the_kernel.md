---
title: Supporting PGP keys and signatures in the kernel
url: https://lwn.net/Articles/882426/
date: "January 25, 2022"
category: Cryptography
author: "By Jake Edge January 25, 2022"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
January 25, 2022

A few weeks back, we [looked](<https://lwn.net/Articles/880263/>) at a proposal to add an integrity-management feature to Fedora. One of the selling points was that the integrity checking could be done using the PGP signatures that are already embedded into the RPM package files that Fedora uses. But the kernel needs to be able to verify PGP signatures in order for the Fedora feature to work. That addition to the kernel has been proposed, but some in the kernel-development community seem less than completely enthusiastic about bringing PGP support into the kernel itself. 

Roberto Sassu [proposed](<https://lwn.net/ml/linux-kernel/20220111180318.591029-1-roberto.sassu@huawei.com/>) the addition of support for PGP keys and signatures based on earlier work by David Howells. Sassu is also proposing the [Digest Lists Integrity Module](<https://lwn.net/ml/linux-kernel/20210914163401.864635-1-roberto.sassu@huawei.com/>) (DIGLIM) for the kernel and is the owner of the [Fedora change proposal](<https://fedoraproject.org/wiki/Changes/DIGLIM>) to support DIGLIM in the distribution. That proposal was originally targeting Fedora 36, but it is not likely to be adopted for any Fedora release until both DIGLIM and PGP support are upstream. 

#### Parser

In order to handle the keys and signatures, though, there is a need for the kernel to understand the [PGP format](<https://datatracker.ietf.org/doc/html/rfc4880>), so Sassu proposed putting a parser into the kernel. As might be guessed, that drew complaints; Maciej S. Szmigiero [wondered](<https://lwn.net/ml/linux-kernel/ab29dd6f-1301-e012-8898-9c739ca511a3@maciej.szmigiero.name/>) if there was a better way: 

> Aren't PGP keys simply RSA / ECC / EdDSA keys with additional metadata? Can't they be unwrapped from their (complex) PGP format in userspace and loaded raw into the kernel, in a similar way as they are sometimes used for SSH authentication? 
> 
> This will save us from having to add complex parsers (a well-known source of bugs) into the kernel [...] 

Sassu [said](<https://lwn.net/ml/linux-kernel/b37f9c0e9bf941f0b778c6949538835d@huawei.com/>) that Howells had already implemented a parser that follows the RFC for PGP data; unwrapping it into another format still requires a parser of some kind. In addition, that code has been tested in several different ways: 

> I extensively tested the implementation with an ad-hoc fault injector, to see if the code can correctly handle errors. I also developed a fuzzer to corrupt the data before it is read by the kernel. Finally, I checked that there are not memory leaks. But I agree, there could still be bugs. 

Meanwhile, an ""in-kernel PGP Web of Trust"", as Szmigiero asked about, would be available to a limited extent, Sassu said. Certain kernel keyrings, such as the [Integrity Measurement Architecture](<https://sourceforge.net/p/linux-ima/wiki/Home/>) (IMA) keyring, require that keys being added to them are vouched for by another key on the kernel keyrings; support for checking the PGP signature on a key and rejecting the key if the check fails is part of what Sassu has implemented in the patch set. 

But Szmigiero was still [concerned](<https://lwn.net/ml/linux-kernel/ab3d2bda-a704-f5d3-adee-e52b7d0a4641@maciej.szmigiero.name/>) about an in-kernel parser: 

> A parser in userspace is preferred to one in kernel since if there is a bug somewhere its consequences are much less severe. And experience shows that parsers are especially prone to bugs. A userspace implementation can also be tightly sandboxed for extra security. 

He also pointed out that there are existing user-space parsers for OpenPGP that could be used, but Sassu does not see that path as less worrisome: 

> However, it does not seem to me less risky to define a new format to upload the RSA key and the signature to avoid the complexity of PGP. Also, it does not seem more complex than [PKCS#7](<https://en.wikipedia.org/wiki/PKCS_7>), which is already in the kernel. 
> 
> [...] Another concern that I have is that, the approach of using an OpenPGP library still requires the Linux distribution vendors to do a transformation from the source data they have to another format. They have anyway to certify this transformation, even if it is done in user space. Maybe it is easier to keep the original data and verify the minimum necessary to handle PGP keys and signature in the kernel, rather than verifying a library running in user space with many other functions. 

#### PGP deprecated?

Jason A. Donenfeld had a [more fundamental concern](<https://lwn.net/ml/linux-kernel/YeV+jkGg6mpQdRID@zx2c4.com/>) about adding PGP support to the kernel at all. ""The general consensus in the crypto engineering world is that PGP ought to be on its way out."" He pointed to a [2019 blog post](<https://latacora.micro.blog/2019/07/16/the-pgp-problem.html>) as an example of the breadth of problems with PGP (and the applications that implement it, such as [GnuPG](<https://gnupg.org/>)). He suggested using an [Ed25519](<https://en.wikipedia.org/wiki/EdDSA>)-based signature mechanism with something like [Minisign](<https://jedisct1.github.io/minisign/>) or [signify](<https://man.openbsd.org/signify>) in user space, and having only minimal code in the kernel; that would have ""very few moving parts to break"", he said. 

But James Bottomley [disagreed](<https://lwn.net/ml/linux-kernel/d92912bba61ee37e42d04b64073b9031604acc0f.camel@HansenPartnership.com/>); while there are legions of PGP detractors, ""no-one's actually been able to come up with a more usable replacement"". Beyond that, Ed25519 is not supported by the [Trusted Platform Module](<https://en.wikipedia.org/wiki/Trusted_Platform_Module>) (TPM) and might never be; by the time the TPM standards organization adds new algorithms, it might skip over Ed25519 for something stronger. He said that the crypto community often wants minimal solutions, but those have ""only a relatively small set of possible use cases and no future proofing"", while more extensible solutions that are more complex get criticized ""as being a 'swiss army knife'"" 

Antony Vennard [thought](<https://lwn.net/ml/linux-kernel/079f10b9-060b-3a36-2224-fa1b483cbad5@vennard.ch/>) that was the wrong way to look at things, though. There is already support for PKCS #7 in the kernel, adding PGP support further complicates things: 

> We already have one extensible system that is complicated in the kernel. This patch proposes to add a second one achieving the same purpose instead of a userspace solution to take advantage of the existing code, which I think would be preferable and safer. 

Sassu [replied](<https://lwn.net/ml/linux-kernel/f42065e2f52a4939b379a8453369aa7b@huawei.com/>) that there are still barriers to the use of alternative mechanisms; whatever key format is adopted will still need some kind of parser and a way to verify signatures on the keys that will need to be present in the kernel. He has considered the problems and believes his approach achieves the right balance: 

> After several iterations, trying to remove anything that was not essential from the patch set, I came to the conclusion that having the logic for both the keys and the signatures was the best compromise. True, it means more maintenance effort for the kernel, but this ensures the best interoperability with the current ecosystem. 
> 
> I think the code is not particularly complicated that would be impractical to review. 

While the PGP format has lots of different packet types, the in-kernel parser ""just needs to parse the key, user ID and the signature packet and subpackets"", he said. The distribution ecosystem is one of his areas of concern with switching to alternative solutions; ""I think that forcing all Linux distributions to do it seems unrealistic"". But if a minimal PGP parser could be vetted, the distribution's use cases would be supported: 

> Parsing data in the PGP format seems to be the only barrier. Once we are confident that we parse this small number of packets correctly, the rest is just reusing the mechanisms already in place. And we make it easier for people to adopt security features. 

He elaborated further on the distribution side of the puzzle in a [reply](<https://lwn.net/ml/linux-kernel/887a1e46cd6f4c02a6530a15f00e8eb8@huawei.com/>) to Donenfeld. Switching to another package-signing mechanism (based on Ed25519 as Donenfeld suggested, say) would require an enormous shift for the distributions, which would likely take years to accomplish: 

> More specifically, the first task would be to modify how RPMs are signed (and thus how they are verified). The second task would be to have a different way to certify the public key. Lastly, Linux distribution vendors would have to change their building infrastructure to use the new certified key, a new version of the rpm package manager which takes as input the new key, produces a different type of signature and embeds it in the RPM header. 
> 
> [...] The aim of this patch set is to make some security features available in a short time, by significantly reducing the burden of Linux distribution vendors for managing those security features. 

Donenfeld sympathized with the situation for distributions, but [noted](<https://lwn.net/ml/linux-kernel/CAHmME9pMEJxGzmetDu0Bkf1=fqb+BHGQ3-6NB3hhtRjLWHm1cA@mail.gmail.com/>) that it comes down to short-term versus long-term thinking; instead of putting a lot of work into a transition to a better scheme (which requires ""evaluating *what* the better thing would be""), continuing to support a flawed (but functioning) system is an easier path forward. He wondered if that kind of thinking needed to come to a halt, at least for PGP: 

> I don't have a whole lot _technical_ to say about long term vs short term thinking, but it does strike me that PGP is one of these cases where people have known about the flaws for decades, but the ecosystem keeps being extended because people continue to go with the short term solutions, one by one, and now they've wound up here, at the doorstep of the kernel. Maybe if at some point somebody puts down the foot and says, "the road of short term intentions stops here," there might gradually be impetus toward looking into long term solutions, e.g. viable PGP replacements? Just a thought. 

While Ed25519 might be attractive, Konstantin Ryabitsev [said](<https://lwn.net/ml/linux-kernel/20220117165933.l3762ppcbj5jxicc@meerkat.local/>), its key management leaves something to be desired: 

> I am concerned that ed25519 private key management is very rudimentary -- more often than not it is just kept somewhere on disk, often without any passphrase encryption. 
> 
> With all its legacy warts, GnuPG at least has decent support for hardware off-load via OpenPGP smartcards or TPM integration in GnuPG 2.3, but the best we have with ed25519 is passhprase protection as implemented in minisign (and even that is rudimentary -- if you need to sign 10 things, you will need to type in your passphrase 10 times, as there is no passphrase agent of any kind). 

He said that the [integration](<https://thehackernews.com/2020/02/openssh-fido-security-keys.html>) of support for [FIDO2](<https://fidoalliance.org/fido2/>) into [OpenSSH](<https://www.openssh.com/>) is the ""most promising non-PGP development of PKI [public-key infrastructure] signatures that I've seen lately"". Key management has been a known weakness for many of the public-key signature schemes over the years. which makes him ""worried that it's going to result in a net loss in overall security"". 

It is less than clear where this proposal will lead at this point. The kernel is the obvious "trusted" place to store keys and verifying signatures made with those keys is a logical extension of that. But the resistance, both to PGP and to another in-kernel parser for a complicated format, may be too much to overcome.
