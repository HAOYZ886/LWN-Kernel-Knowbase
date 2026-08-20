---
title: Opening up kernel security bug handling
url: https://lwn.net/Articles/566123/
date: "September 11, 2013"
category: Security
author: "By Jake Edge September 11, 2013"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
September 11, 2013

The reporting and handling of security issues is a tricky proposition. There are numerous competing interests to try to balance, and a general tendency toward secrecy that can complicate things further. Thus it is not surprising that kernel developers are discussing security handling on the [Kernel Summit](<http://events.linuxfoundation.org/events/linux-kernel-summit>) discussion mailing list ([ksummit-2013-discuss](<https://lists.linuxfoundation.org/mailman/listinfo/ksummit-2013-discuss>)). It seems likely that discussion will pick up again at the summit itself, which will be held in Edinburgh, October 23-25. 

James Bottomley kicked off the discussion by [noting](<http://lists.linuxfoundation.org/pipermail/ksummit-2013-discuss/2013-August/001050.html>) that several recent fixes had gone into the kernel without following the normal process because they were "security fixes". Given that some of those fixes [caused problems](<https://lwn.net/Articles/562294/#security>) of various sorts, he is concerned about circumventing the process simply because the patches fix security issues: 

In both cases we had commits with cryptic messages, little explanation and practically no review all in the name of security. 

Our core processes for accepting code require transparency, review and testing. Secrecy in getting code into the kernel is therefore fundamentally breaking this and risking the kinds of problems we see in each of the instances. 

Bottomley would like to explore whether security vulnerabilities need to be handled in secret _at all_. Given that he thinks that may not be popular, looking into what can be done to inject more transparency into the process would be a reasonable alternative. Part of his theory is that ""security people"" who ""love secrecy"" are running the vulnerability-handling process. 

For example, the closed kernel security mailing list (security@kernel.org) is either made up of "security officers" (according to `Documentation/SecurityBugs`) or ""'normal' kernel developers"" ([according to Greg Kroah-Hartman](<http://lists.linuxfoundation.org/pipermail/ksummit-2013-discuss/2013-August/001054.html>)). There is no inherent interest in secrecy by the participants on that list, Kroah-Hartman said, though he did agree that posting a list of the members of security@kernel.org—which has not yet happened—would help to make things more transparent. The relationship between the kernel security list and the linux-distros mailing list (a closed list for distribution security concerns—the successor to vendor-sec) is also a bit murky, which could use some clearing up, Bottomley said. 

A big part of the problem is that there are a few different constituencies to try to satisfy, including distributions (some of which, like enterprise distributions, may have additional needs or wants), users (most of whom get their kernel from a distributor or device maker), security researchers (who sometimes like to make a big splash with their findings), and so on. While it might be tempting to dismiss the security researchers as perpetrators of what Linus Torvalds likes to call "the security circus", it is important to include them. They are often the ones who find vulnerabilities; annoying them often results in them failing to report what they find, sadly. 

Secrecy in vulnerability handling may be important to the enterprise distributions for other reasons, as Stephen Hemminger [said](<http://lists.linuxfoundation.org/pipermail/ksummit-2013-discuss/2013-August/001056.html>). Security vulnerabilities and response time are often used as a "sales" tool in those markets, so that may lead to a push for more secrecy: 

It seems to me that the secrecy is more about avoiding sensationalist news reports that might provide FUD to competitors. For the enterprise products this kind of FUD might impact buying decisions and even the financial markets. 

Torvalds's [practice](<https://lwn.net/Articles/476947/#fix>) of hiding the security implications of patches also plays a role here. He wants to mask vulnerabilities so that "black hats" cannot easily `grep` them from commit logs, but as James Morris [pointed out](<http://lists.linuxfoundation.org/pipermail/ksummit-2013-discuss/2013-August/001055.html>), that's not really effective: ""The cryptic / silent fixes are really only helping the bad guys. They are watching these commits and doing security analysis on them."" 

It seems unlikely (though perhaps not completely impossible) that Torvalds would change his mind on the issue, so various ideas on collecting known security information correlated with the commit(s) that fixed them were batted around. Clearly, some information about security implications only comes to light after the commit has been made—sometimes long after—so there is a need to collect it separately in any case. 

Kees Cook [described](<http://lists.linuxfoundation.org/pipermail/ksummit-2013-discuss/2013-August/001074.html>) some of the information that could be collected, while Andy Lutomirski [expanded on the idea](<http://lists.linuxfoundation.org/pipermail/ksummit-2013-discuss/2013-August/001075.html>) by suggesting separate CVE files stored in the kernel tree. The idea seemed fairly popular; others chimed in with suggestions for collaborating with Debian and/or the linux-distros mailing list participants. In a separate sub-thread, Lutomirski [created a template](<http://lists.linuxfoundation.org/pipermail/ksummit-2013-discuss/2013-August/001337.html>) for how the information could be stored. Cook [concurred](<http://lists.linuxfoundation.org/pipermail/ksummit-2013-discuss/2013-August/001339.html>) and suggested that the files could live under `Documentation/CVEs` or something similar. It is clear that there is an interest in having more data available on security vulnerabilities and fixes in the kernel, so that could lead to a lively discussion in October. 

Some seem to have already started down the path of more openness in the security reporting realm. Lutomirski recently publicly [posted](<https://lwn.net/Articles/566274/>) a fix that was clearly marked as a security fix from the outset. Cook did much the same with a [list of vulnerabilities](<https://lwn.net/Articles/566276/>) in the kernel's human interface device (HID) code. Exploiting the HID bugs requires physical access and specialized devices, but that may be part of the threat model for certain users. These aren't the first reports of this kind; others have been made from time to time. In fact, certain subsystems (networking, in particular) essentially never use the closed list and prefer to work on security problems and fixes in the open. 

An even more recent example comes from Wannes Rombouts's [report](<https://lwn.net/Articles/566277/>) of a networking security hole (use after free), which was referred to the netdev mailing list by security@kernel.org. The implications of the bug were not completely clear (either to Rombouts or to Hemminger, who [replied](<https://lwn.net/Articles/566279/>)), but Ben Hutchings [recognized](<https://lwn.net/Articles/566281/>) that user namespaces could make the problem more widespread (when and if they are enabled in most kernels anyway). Though it is networking related—thus the referral to netdev, presumably—this is the kind of vulnerability that could have been handled behind closed doors. But because it was posted to an open list, the full implications of the problem were discovered. In addition, for this bug (as well as for Lutomirski's and Cook's bugs), those affected have the ability to find out about the problems and either patch their kernels or otherwise mitigate the problem. And _that_ is another advantage of openness.
