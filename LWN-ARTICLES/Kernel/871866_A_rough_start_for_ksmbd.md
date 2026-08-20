---
title: A rough start for ksmbd
url: https://lwn.net/Articles/871866/
date: "October 7, 2021"
category: "Filesystems-ksmbd; Releases-5.15"
author: "By Jonathan Corbet October 7, 2021"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
October 7, 2021

Among the many new features pulled into the mainline during the 5.15 merge window is the [ksmbd](<https://github.com/namjaejeon/ksmbd>) network filesystem server. Ksmbd implements the [SMB protocol](<https://en.wikipedia.org/wiki/Server_Message_Block>) (also known as CIFS, though that name has gone out of favor) that is heavily used in the Windows world. The creation of an in-kernel SMB server is a bit surprising, given that Linux has benefited greatly from the user-space [Samba](<https://www.samba.org/>) solution since shortly after the beginning. There are reasons for this move but, in the short term at least, they risk being overshadowed by a worrisome stream of security-related problems in ksmbd. 

Why create an in-kernel SMB server at this point? In a sense, ksmbd is not meant to compete with Samba; indeed, it has been developed in cooperation with the Samba project. It is, however, meant to be a more performant and focused solution than Samba is; at this point, Samba includes a great deal of functionality beyond simple file serving. Ksmbd claims significant performance improvements on a wide range of benchmarks; the [graphs on this page](<https://github.com/namjaejeon/ksmbd#performance>) show a doubling of performance on some tests. An in-kernel server is an easier place to support variants like [SMB Direct](<https://docs.microsoft.com/en-us/windows-server/storage/file-server/smb-direct>), which uses RDMA to transfer data between systems. By drawing more eyes to the code, merging into the mainline may also facilitate faster development in general. One other reason — which tends to be spoken rather more quietly — is that a new implementation can be licensed under GPLv2, while Samba is GPLv3. 

Ksmbd was first [posted for review](<https://lwn.net/ml/linux-kernel/20210322051344.1706-1-namjae.jeon@samsung.com/>) (as "cifsd") by Namjae Jeon in late March; [the eighth revision](<https://lwn.net/ml/linux-kernel/20210823025816.7496-1-namjae.jeon@samsung.com/>) came out just before the opening of the 5.15 merge window in late August. The last version received no review comments, but previous versions had clearly been looked at by a number of developers. Nobody objected when Steve French [asked Linus Torvalds](<https://lwn.net/ml/linux-kernel/CAH2r5msoKV7qAgoKipa%2BQNDJ%2BxR83YGuz%2Bhe%2BGH9sPTSzBMLHA%40mail.gmail.com/>) to pull ksmbd into the mainline on August 29\. 

It is not unusual for a new subsystem to receive a lot of fixes after its entry into the mainline kernel. Merging tends to draw a lot more attention to the code, and the number of testers normally goes up, leading to the discovery of more problems. That is what the stabilization period after the merge window is for, after all. That said, the nature of the fixes being applied can give some insight into the quality of the underlying code, and the indications for ksmbd are not entirely good. 

The [commit history for ksmbd](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/fs/ksmbd>) shows a steady stream of fixes, as expected. Worryingly, though, many of the problems being fixed are clearly security issues — not a good thing in a network filesystem implementation. Examples include: 

  * The code to change ownership and permissions [did not check existing file permissions](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/fs/ksmbd?id=e70e392fa768d46ca59f2f8c0e7374099c980622>) first. 
  * [Failure to validate data lengths](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/fs/ksmbd?id=303fff2b8c77a85c62dbde3b27c24b084144c04c>) could lead to access to invalid data. 
  * The server would [blindly follow symbolic links](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/fs/ksmbd?id=4ea477988c423a57241ea4840b12832de6fabdfd>) during pathname lookup. 
  * Numerous failures to validate buffer lengths, such as [this one](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/fs/ksmbd?id=d72a9c158893d537d769a669a5837bc80b0f851c>) or [this one](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/fs/ksmbd?id=9496e268e3af78a92778bf635488a8ec2dca8996>). 

All of those fixes were applied after ksmbd landed in the mainline; there are others that came before. Currently, twelve fixes to ksmbd credit Coverity scans in their changelogs. 

Again, it would not be surprising for a security issue or three to turn up in a new network-filesystem implementation. But ksmbd has shown enough problems to have raised a few eyebrows in the kernel community, though the discussion of those problems was evidently held in private for some time. When French [pushed another set of ksmbd fixes](<https://lwn.net/ml/linux-kernel/CAH2r5mvu5wTcgoR-EeXLcoZOvhEiMR0Lfmwt6gd1J1wvtTLDHA@mail.gmail.com/>) in mid-September, though, Kees Cook [took the discussion public](<https://lwn.net/ml/linux-kernel/202109221850.003A16EC1@keescook/>): 

> I was looking through the history[[1](<https://git.kernel.org/pub/scm/linux/kernel/git/next/linux-next.git/log/fs/ksmbd>)] of the ksmbd work, and I'm kind of surprised at some of the flaws being found here. This looks like new code being written, too, I think (I found[[0](<https://lore.kernel.org/lkml/20210322051344.1706-1-namjae.jeon@samsung.com/>)])? Some of these flaws are pretty foundational filesystem security properties[[2](<https://git.kernel.org/linus/f58eae6c5fa882d6d0a6b7587a099602a59d57b5>)] that weren't being tested for, besides the upsetting case of having buffer overflows[[3](<https://git.kernel.org/linus/6d56262c3d224699b29b9bb6b4ace8bab7d692c2>)] in an in-kernel filesystem server. 
> 
> I'm concerned about code quality here, and I think something needs to change about the review and testing processes. 

French [replied](<https://lwn.net/ml/linux-kernel/CAH2r5muNG4GvziyMG2unkYNjUiT4V+pz0pWUGkWQNxUZJnBadw@mail.gmail.com/>) that he was surprised by some of the problems too. He pointed to [a wiki page](<https://wiki.samba.org/index.php/Ksmbd-review>) describing the ongoing security review for this code, which seems to have acquired a new urgency. A number of new procedures are being instituted, he said, and there will be testing done at various interoperability events. French said he was ""pleased with the progress that is being made, but also conceded that ksmbd ""is not ready for production use yet"". 

There are also some things to look forward to on the security front, he continued: 

> There is some good news (relating to security), once Namjae et al get past these buffer overflow etc. patches. 
> 
>   * he has already implemented the strongest encryption supported in SMB3.1.1 
>   * he has implemented the man in the middle attack prevention features of the protocol 
>   * strong (Kerberos) authentication is implemented 
>   * he has removed support for weak older dialects (including SMB1 and SMB2) of the protocol 
>   * he will be removing support for weaker authentication (including NTLMv1) 

The NTLMv1 removal has [since been merged](<https://git.kernel.org/pub/scm/linux/kernel/git/next/linux-next.git/commit/fs/ksmbd?id=ce812992f239f45e13c820a52455fec6eacbce1e>) into the mainline. On reading French's message, Cook [responded](<https://lwn.net/ml/linux-kernel/202109231109.0AD3D5A@keescook/>): ""Thanks for making these recent changes; I feel much better about ksmbd's direction"". 

The work on cleaning up ksmbd proceeds; French [pushed another 11 fixes](<https://lwn.net/ml/linux-kernel/CAH2r5muy3GtTQPoaVXiD_tU-cG4FAQk4SCmmiR8vS4_pWvPanw@mail.gmail.com/>) on October 1\. 

At this point, there is little doubt that ksmbd will be properly reviewed and cleaned up; there are eyes on the code, and ksmbd itself is small enough that a comprehensive review should be feasible. At that point, the kernel should have an SMB implementation that is feature-rich, performant, and secure. That said, waiting another kernel development cycle or two for the developers to ""get past these buffer overflow etc. patches"" before deploying it might well be prudent. 

This is all good, but it is still a little worrisome that this code got as far as it did in the condition it was in. It seems clear that security concerns were not at the forefront when this code was being developed and that the review it received before being merged failed in this regard as well. The addition of security features is great, but they do not help much in the absence of a secure implementation. If we ever want to reach a point where we are not adding more security problems to the kernel than we are fixing, we will need to do better than this.
