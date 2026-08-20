---
title: Kernel security reporting for distributions
url: https://lwn.net/Articles/941745/
date: "August 16, 2023"
category: "Development model-Security issues"
author: "By Jake Edge August 16, 2023"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
August 16, 2023

The [call for topics](<https://lwn.net/Articles/941660/>) for the [Linux Kernel Maintainers Summit](<https://events.linuxfoundation.org/linux-kernel-maintainer-summit/>) went out on August 15; one proposed topic has generated some interesting discussion about security-bug reporting for the kernel. A recent [patch](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=4fee0915e649b>) to the kernel's documentation about how to report security bugs recommends avoiding posting to the [linux-distros mailing list](<https://oss-security.openwall.org/wiki/mailing-lists/distros>) because its goals and rules do not mesh well with kernel security practices. That led Jiri Kosina to [suggest a discussion on security reporting](<https://lwn.net/ml/ksummit-discuss/nycvar.YFH.7.76.2308150927190.14207@cbobk.fhfr.pm/>), especially with regard to Linux distributions. 

The linux-distros mailing list is a closed list for reporting security bugs that affect Linux systems; as might be guessed, the participants are representatives of various distributions. It has some fairly stringent requirements regarding the maximum embargo period (14 days) after a bug is reported before it must be publicly disclosed; it also places requirements on the reporter to post the full details to the [oss-security mailing list](<https://oss-security.openwall.org/wiki/mailing-lists/oss-security>) once the embargo has run its course. These policies have clashed with kernel bug reporting along the way. Examples include a [2018 embargo that went awry](<https://lwn.net/Articles/762512/>), an [even longer embargo botch](<https://lwn.net/Articles/874069/>) in 2021, and a [2022 discussion of the problems](<https://lwn.net/Articles/896829/>). 

The core of the documentation patch from Greg Kroah-Hartman changes a suggestion that some bugs might need to be coordinated with linux-distros to a strongly worded admonition against doing so: 

> The kernel security team strongly recommends that reporters of potential security issues NEVER contact the "linux-distros" mailing list until AFTER discussing it with the kernel security team. Do not Cc: both lists at once. You may contact the linux-distros mailing list after a fix has been agreed on and you fully understand the requirements that doing so will impose on you and the kernel community. 
> 
> The different lists have different goals and the linux-distros rules do not contribute to actually fixing any potential security problems. 

Kosina said that he generally agreed with the change, but wanted to discuss how to report kernel security bugs to the distributions in some other fashion. In part, he was looking at the problem from the perspective of a distribution maintainer for SUSE: 

> With my distro hat on, I really want the kernel security bugs to be *eventually* processed through linux-distros@ somehow, for one sole reason: it means that our distro security people will be aware of it, track it internally, and keep an eye on the fix being ported to all of our codestreams. This is exactly how this is done for all other packages. 
> 
> I would be curious to hear about this from other distros, but I sort of expect that they would agree. 
> 
> If this process doesn't happen, many kernel security (with CVE potentially eventually assigned retroactively) fixes will go by unnoticed, as distro kernel people will never be explicitly made aware of them, and distros will be missing many of the patches. Which I don't think is a desired end effect. 

Kosina noted that the advice from Kroah-Hartman is that the distributions should simply follow one of the stable trees, but that is unworkable for various reasons. One concrete problem that SUSE has is tracking whether a given CVE has been fixed in its kernel; even though he believes that the CVE system is seriously flawed, tracking CVEs and ensuring the fixes get into SUSE kernels is critical, especially for certain kinds of customers. The linux-distros list provided that tracking in many cases ""and that is lost if the fix goes only through security@kernel.org"". 

Vegard Nossum [agreed](<https://lwn.net/ml/ksummit-discuss/658e739b-c164-c360-d6a3-eb4fb15ae02e@oracle.com/>) that the topic needed discussion and pointed to his [earlier efforts to overhaul the security-reporting documentation](<https://lwn.net/ml/linux-kernel/20230305220010.20895-1-vegard.nossum@oracle.com/>). It would make sense to discuss the issue at the summit in part because the recent documentation change came about due to complaints from Linus Torvalds on the closed lists, he said. ""I therefore think that Linus himself needs to be involved in this discussion and that his arguments need to be made in public instead of just paraphrased by me."" In part, Torvalds was unhappy with requirements that proof-of-concept code exploiting a vulnerability has to be published if the code was posted to linux-distros. 

#### distros@kernel.org?

Nossum suggested having a closed list for distribution representatives that was run by the kernel security team. The list's policies and membership could be set by the team and the list could be used to post patches for security fixes once they are ready for the distributions to incorporate them. Kroah-Hartman was quick to [throw cold water on that idea](<https://lwn.net/ml/ksummit-discuss/2023081515-lake-spotty-6a3a@gregkh/>), however. 

A list that pre-announces security problems will either be ""deemed illegal in some countries"" or be required to admit all "major" Linux users, which would include various government agencies across the globe, he said. For that reason, the Linux Foundation is not going to be willing to host such a list and he suspects it would be hard to find an organization willing to do so; ""it's amazing that linux-distros has been able to survive for as long as it has"". 

Steven Rostedt [wondered](<https://lwn.net/ml/ksummit-discuss/20230815084253.7091083e@gandalf.local.home/>) if there were ways to solve Kosina's problem without running afoul of concerns over pre-announcements. He noted that the distribution kernels are the ones that are in the hands of users, so it would be best to ensure that the maintainers of those kernels have access to the information about security flaws. But Kroah-Hartman [sees the landscape rather differently](<https://lwn.net/ml/ksummit-discuss/2023081540-vindicate-caterer-33c6@gregkh/>): 

> The huge majority of Linux use in the world is Android, everything else is a rounding error. Android does now have projects where security bugs reported to them are required for the reporter to submit the problem to security@k.o as well. We fix the issue, get it pushed out, and the reporter gets the credit. [...] 
> 
> After Android, Debian is by far the largest Linux user, and the Debian kernel developers do an awesome job of tracking the stable kernel releases which have been documented to fix 99% of known security issues _BEFORE_ they are known (data produced by Google security team for 2 years straight). 
> 
> After that, the use of Linux tapers off to "roll your own kernel.org releases" (still a huge number of absolute boxes thanks to many government instances and corporate clouds) to the various "enterprise" distros, down to the other community distros (Fedora/openSUSE/Arch/etc.) 

He lamented the length of time it takes for Android to actually get these fixes in the hands of its users (4-6 months), but praised the fact that Android follows the stable releases. He believes that governments are moving toward regulations that will reduce those delays in any case. The distributions that are not following the stable kernels are generally those for enterprise Linux: 

> It's that corporate sponsored "middle tier" (which is still quite small overall) that likes complaining about this stuff because they don't want to take the stable kernel updates for various (in my opinion stupid) reasons. 
> 
> So who would such a "distros@" list help? Only that middle-tier, small group of very well financed companies that want to go their own way with their kernel releases because they want to. 

Kosina [called that an oversimplification](<https://lwn.net/ml/ksummit-discuss/nycvar.YFH.7.76.2308160005300.14207@cbobk.fhfr.pm/>), noting that Android is huge in terms of users, but that the customers for the enterprise distributions ""directly contribute back to kernel development, by allowing companies like Red Hat, SUSE, Canonical, ... to put kernel engineers on their payroll"". Rostedt [agreed](<https://lwn.net/ml/ksummit-discuss/20230815183120.0c92a759@gandalf.local.home/>), noting that the piecemeal impact of vulnerabilities in Android kernels is rather different than that of such flaws for enterprise kernels: ""All it takes is to go after one server, and you have access to thousands of users and their private data."" 

Kroah-Hartman [walked back](<https://lwn.net/ml/ksummit-discuss/2023081641-unstitch-kangaroo-a2c1@gregkh/>) his "rounding error" comment to a certain extent, but wanted to focus on the ""false sense of security"" that comes from the enterprise distributions picking and choosing fixes to apply to their kernels. The Linux community finds and fixes every bug that it can; it provides those kernels to the world but they do not reach all users because of the decisions made by certain companies. Meanwhile: ""those 'policy decisions of companies' are now known by governments to be incorrect, and soon will be made illegal in many countries, so we are on the right side here."" 

#### Working with linux-distros?

The linux-distros (and oss-security) mailing lists are administered and moderated by Alexander "Solar Designer" Peslyak, who was alerted to the discussion by Nossum. Peslyak [posted](<https://lwn.net/ml/ksummit-discuss/20230816152621.GA8252@openwall.com/>) a lengthy response, suggesting that interested people could perhaps meet at [Open Source Summit Europe](<https://events.linuxfoundation.org/open-source-summit-europe/>) since he would not be able to be present at the Maintainers Summit. He said that email discussion would work, perhaps better in fact, ""but meeting in person is a gesture that might help establish an atmosphere of trust and assurance of good intent"". 

He pointed out that the new wording in the documentation does not _preclude_ posting to linux-distros, but says that it should happen only in consultation with the kernel security team. He does not know if that would actually happen, but: ""Maybe some friendly dialogue and agreeing on things could affect that."" There seem to be two problem areas: the linux-distros disclosure deadline of 14 days even if there is no fix yet and the requirement that any proof-of-concept posted to the list be made public within seven days of the vulnerability disclosure. He agreed that both are problems and suggested that a discussion on oss-security might lead to ""satisfactory solutions/exceptions"". 

It seems clear that Kroah-Hartman, at least, does not see a real problem to be solved. As he [pointed out](<https://lwn.net/ml/ksummit-discuss/2023081512-worshiper-donor-371f@gregkh/>), though, the security team is ""run as a 'collective' by those doing the work there, not by fiat"". So perhaps others will be inclined to see what can be worked out in order to keep linux-distros in the loop, though it seems like an uphill battle. 

As Peslyak noted, another summit topic post about [quality standards for embargoed fixes](<https://lwn.net/ml/ksummit-discuss/ZNuuvS5BtmjcazIv@sashalap/>) is also worth a look. It would seem that some interesting discussion topics are already lining up for the summit; look for coverage of whichever topics get chosen, in November, shortly after the event.
