---
title: "Improving the handling of embargoed hardware-security bugs"
url: https://lwn.net/Articles/769417/
date: "October 25, 2018"
category: "Development model-Security issues; Security-Meltdown and Spectre"
author: "By Jonathan Corbet October 25, 2018 Maintainers Summit"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
October 25, 2018

* * *

[Maintainers Summit](<https://lwn.net/Articles/769260/>)

Jiri Kosina kicked off a session on hardware vulnerabilities at the 2018 Kernel Maintainers Summit by noting that there are few complaints about how the kernel community deals with security issues in general. That does not hold for Meltdown and Spectre which, he said, had been "completely mishandled". The subsequent handling of the [L1TF vulnerability](<https://lwn.net/Articles/762570/>) suggests that some lessons have been learned, but there is still plenty of room for improvement in how hardware vulnerabilities are handled in general. 

There are a number of reasons why the handling of Meltdown and Spectre went bad, he said, starting with the fact that the hardware vendors simply did not know how to do it right. They didn't think that the normal security contact (`security@kernel.org`) could be used, since there was no non-disclosure agreement (NDA) in place there. Perhaps what is needed is the creation of such an agreement or, as was [discussed in September](<https://lwn.net/Articles/764593/>), a "gentleman's agreement" that would serve the same role. 

James Bottomley asserted that not even the gentleman's agreement would be needed if the community were to publish a comprehensive document on how it will handle reports of hardware security issues, but others said that the [![\[Jiri Kosina\]](https://static.lwn.net/images/conf/2018/ms/JiriKosina-sm.jpg)](<https://lwn.net/Articles/769427>) problems go beyond the initial agreement. Linus Torvalds complained that he has been unable to get either emails or PDF documents describing known vulnerabilities; all that has been on offer is the ability to get an account on an Intel server where documents can be read. Thomas Gleixner said that there has been some progress in that area, though, and that he is now able to get documents in a GPG-encrypted tarball. 

Greg Kroah-Hartman said that the wording of [the documentation on how security issues are handled](<https://www.kernel.org/doc/html/v4.19/admin-guide/security-bugs.html>) is not perfect for this case, but work is being done to fix it. Gleixner said that we need to create a single point of contact for hardware vulnerabilities; the vendors will then understand the rules that we play by and that we will not leak information. Intel, he said, has learned a lot and knows who to talk to. Mauro Carvalho Chehab complained that Intel is just one vendor, though, and that the next vendor with a vulnerability will be different. Torvalds replied that the most important vendors are coming around; Gleixner added that this is another reason to have clear documentation on how we have handled these problems in the past. 

Ted Ts'o said that the community's policy is to hold on to fixes for kernel bugs for up to five working days while distributors work out their response. That time period is clearly not appropriate for hardware bugs, but what would the right time be? Gleixner responded that it is "quite long". Vendors can come up with a proof-of-concept microcode update for a single product fairly quickly, but that is just the beginning; vendors like Intel have hundreds of products, each of which must be evaluated and fixed independently. So the response time tends to drag out; the kernel community has to acknowledge that hardware vendors need time to handle things properly. 

Kees Cook asked how long that would be, but it seems that the answer varies considerably depending on the nature of the vulnerability. The L1TF fixes were ready three weeks before the disclosure, helped by the fact that Intel had informed the community even before it knew how many processors were affected. Torvalds complained, though, that many of the embargo periods are still controlled by "the old corrupt security rules"; the L1TF disclosure date was determined by the date of a security-conference talk rather than any technical considerations. That is not a game we want to play, he said. 

Cook persisted, asking whether the community could somehow set a maximum embargo time. Gleixner said that would be difficult. We can't create our own patches before any microcode fixes are done, for example. There are also delays associated with the interaction with other operating-system vendors, some of whom are slower than the Linux community to prepare patches. Those vendors, Kosina said, have venues where they are able to collaborate on issues like these, but the kernel is not represented there. Gleixner said that the community needs a contact point that can participate in these discussions. 

Torvalds said that the hardware vendors worry a lot that problems will not be kept under wraps until the appointed disclosure day; they need to have personal connections with the community to get over those fears. Gleixner agreed, saying that a new contact point should be set up for hardware issues; it would be a smaller group than `security@kernel.org`. The vendors would have to trust that group, though, and would have to allow domain experts to be brought in from outside the group for specific problems. Extending that trust or not is their decision in the end, he said; if they won't play, then Linux will simply wait until the issues become public to start work on fixing them. 

Will Deacon said that, if one vendor has a specific type of problem, others probably do as well; there's only so much novelty in the area of microprocessor design. But hardware vendors don't have a way to coordinate around this kind of vulnerability; indeed, they tend to do the opposite. If a group of developers is talking to one hardware vendor, the other vendors will stay away from that group. That implies, he said, that the point of contact for each processor type needs to be the associated architecture maintainer. Kroah-Hartman agreed, saying that the cross-vendor collaboration problems are not amenable to solution by the Linux kernel community. 

Arnd Bergmann asked about the problem of older, unmaintained processors. A number of old MIPS processors are affected, for example, but nobody is doing anything about them. Kroah-Hartman said there is little for the community to do about abandoned hardware; that is an issue for governments to deal with. Until the ability to update hardware and ongoing security support are mandated, the problem will persist. 

As the session concluded, Grant Likely said that the community needs to develop a documented process for hardware vulnerabilities — before the next one hits. But who would write this document? After an awkward silence, offers of help were received from Deacon, Gleixner, Kosina, Kroah-Hartman, and Likely, with your editor being instructed by the rest to ensure that all of the names were written down and published. 

[Thanks to the Linux Foundation, LWN's travel sponsor, for supporting my travel to the Maintainers Summit.]
