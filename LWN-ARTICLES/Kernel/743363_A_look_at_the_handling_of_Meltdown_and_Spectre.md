---
title: A look at the handling of Meltdown and Spectre
url: https://lwn.net/Articles/743363/
date: "January 9, 2018"
category: "Security-Meltdown and Spectre"
author: "By Jake Edge January 9, 2018"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jake Edge**  
January 9, 2018

The [Meltdown/Spectre](<https://spectreattack.com/>) debacle has, deservedly, reached the mainstream press and, likely, most of the public that has even a remote interest in computers and security. It only took a day or so from the accelerated disclosure date of January 3—it was originally scheduled for January 9—before the bugs were making big headlines. But Spectre has been known for at least six months and Meltdown for nearly as long—at least to some in the industry. Others that were affected were completely blindsided by the announcements and have joined the scramble to mitigate these hardware bugs before they bite users. Whatever else can be said about Meltdown and Spectre, the handling (or, in truth, mishandling) of this whole incident has been a horrific failure. 

For those just tuning in, Meltdown and Spectre are two types of hardware bugs that affect most modern CPUs. They allow attackers to cause the CPU to do speculative execution of code, while timing memory accesses to deduce what has or has not been cached, to disclose the contents of memory. These disclosures can span various security boundaries such as between user space and the kernel or between guest operating systems running in virtual machines. For more information, see the [LWN article on the flaws](<https://lwn.net/Articles/742702/>) and the [blog post](<https://www.raspberrypi.org/blog/why-raspberry-pi-isnt-vulnerable-to-spectre-or-meltdown/>) by Raspberry Pi founder Eben Upton that well describes modern CPU architectures and speculative execution to explain why the Raspberry Pi is not affected. 

Given the nature of the bugs, and that cache-timing side channels have been known for some time, it is a bit surprising that these longstanding flaws were all found relatively recently. In fact, it would appear that they were all found by independent teams (three for Meltdown and two for Spectre) at more or less the same time. That worryingly suggests that it is not outside the realm of possibility that black-hat attackers and/or government spy agencies may have gotten there first—maybe by years. There is no evidence of that occurring, but attacks using the flaws would not be easy to detect, though using the information disclosed might have set off some alarm bells. 

#### Discovery and disclosure

Jann Horn of Google's Project Zero discovered both of the flaws as part of his research; he [detailed them in a lengthy blog post](<https://googleprojectzero.blogspot.com/2018/01/reading-privileged-memory-with-side.html>). In that post, he noted that Spectre was disclosed to Intel, AMD, and ARM on June 1, but that Meltdown was only reported later. As Jan Wildeboer's extensive [timeline](<https://plus.google.com/+jwildeboer/posts/jj6a9JUaovP>) shows, Meltdown was found by Horn (and two academic teams) sometime before the July 28 publication of "[Negative Result: Reading Kernel Memory From User Mode](<https://cyber.wtf/2017/07/28/negative-result-reading-kernel-memory-from-user-mode/>)" by Anders Fogh, which described the Meltdown technique. From the just-released [Project Zero bug report](<https://bugs.chromium.org/p/project-zero/issues/detail?id=1272>), which has lots of good information and proof-of-concept code, we see that Horn refers to "variant 3" (which is Meltdown) on June 22\. 

After those disclosures, Intel (and perhaps the other CPU vendors) started alerting some of its customers and other interested parties under a non-disclosure agreement (NDA). For Ubuntu, that happened on November 9, according to [its timeline](<https://wiki.ubuntu.com/SecurityTeam/KnowledgeBase/SpectreAndMeltdown>). Other Linux distributions were also alerted, though it seems possible that some distributions and, probably, cloud providers were notified earlier. Apple and Microsoft may well have gotten earlier notice too; Apple released fixes for Meltdown on several of its operating systems in mid-December. Since there is, as yet, no timeline from Intel or the other CPU vendors, we are left guessing on who got notified and when. 

We do know that many Linux distributions were left out in the cold, as was the BSD community. [Smaller cloud companies](<https://blog.linode.com/2018/01/03/cpu-vulnerabilities-meltdown-spectre/>) and others with less clout were left out as well—multiple tier-2 cloud providers have [formed a group](<https://techcrunch.com/2018/01/06/how-tier-2-cloud-vendors-banded-together-to-cope-with-spectre-and-meltdown/>) to help cope with the fallout. As might be guessed, that has left some less than entirely impressed with the whole process. 

#### KAISER and KPTI

The bugs were embargoed, but that started to break down as hints about the flaws were published. Before Fogh's post, Daniel Gruss and others at Graz University of Technology published a [paper [PDF]](<https://gruss.cc/files/kaiser.pdf>) that described ways to break kernel address-space layout randomization (KASLR). It also described a mechanism to avoid those problems, called KAISER, that would unmap kernel memory before entering user-space code. A [patch set](<https://lwn.net/Articles/737940/>) to implement KAISER was posted to the linux-kernel mailing list on October 31; it was based on the work by Gruss and his team and forward-ported by Dave Hansen. 

Those patches were undoubtedly not meant as a Halloween gift to security researchers, but may well have served that purpose. Normally, a patch set that radically changed the memory layout of the running kernel at a fairly high performance cost—ostensibly simply to avoid KASLR breaks—would find tough sledding, but something was clearly different with KAISER. After all, it has never been particularly difficult for attackers to break KASLR and preventing that has never been a "drop everything and fix it" kind of problem. But KAISER was treated as an urgent fix. 

The KAISER patches fairly quickly morphed into [kernel page-table isolation](<https://lwn.net/Articles/741878/>) (KPTI) and were [merged](<https://lwn.net/Articles/742404/>) for the 4.15-rc6 release, which is unprecedented in recent times; it was also picked up for the 4.14 stable kernel series. Clearly, something important was afoot. That led to lots of discussion about what the real bug is, here and elsewhere. A [patch](<https://lwn.net/Articles/743743/>) from Tom Lendacky to turn off KPTI for AMD processors led to speculation about speculation (that is, speculative execution). KPTI addressed the Meltdown vulnerability, but Spectre was "unknown" outside of certain rarefied circles until the dam broke on January 3\. 

When that happened, it could plainly be seen that the [state of the mitigations for Spectre](<https://lwn.net/Articles/743265/>) was far behind KPTI. There were multiple patches by authors at different companies, some that didn't apply to any known kernel tree, some that didn't compile, and so on. It was clearly the result of the embargo; instead of working together, various organizations were working on their own. The Spectre flaws are definitely harder to mitigate, but not coordinating the thinking and fixes certainly has not helped matters. So far, no fixes for Spectre have made it into the mainline. One would guess that will change shortly after the release of 4.15, which should come in mid-late January. 

#### Grumbling

There has been a fair amount of grumbling about how this process has played out. Without pointing fingers, Greg Kroah-Hartman [stated his opinion](<http://kroah.com/log/blog/2018/01/06/meltdown-status/>) in a status update on the bug fixes: 

As for how this was all handled by the companies involved, well this could be described as a textbook example of how _NOT_ to interact with the Linux kernel community properly. The people and companies involved know what happened, and I'm sure it will all come out eventually, but right now we need to focus on fixing the issues involved, and not pointing blame, no matter how much we want to. 

But it is clear that at least parts of the Linux kernel community were aware of Meltdown, at least, as far back as the original posting of the KAISER patches. Other open-source operating systems got no warning at all, other than perhaps some background rumbling because of KAISER/KPTI. As OpenBSD developer Philip Guenther [put it](<https://lwn.net/Articles/743676/>): 

We have received *no* non-public information. I've seen posts elsewhere by other *BSD people implying that they receive little or no prior warning, so I have no reason to believe this was specific to OpenBSD and/or our philosophy. Personally, I do find it....amusing? that public announcements were moved up after the issue was deduced from development discussions and commits to a different open source OS project. Aren't we all glad that this was under embargo and strongly believe in the future value of embargoes? 

Meanwhile, Linux distributors have been working up their fixes, which typically need to be done to older kernels, especially for the enterprise distributions. Because of the lead time needed for QA and the like, enterprise distributions have backported the KAISER work into their kernels, rather than the more recent KPTI work. That led Kroah-Hartman to follow their lead for the 4.4 and 4.9 stable trees in order to try to be "bug for bug compatible" with the enterprise distributions. There are, it seems, lots of users of kernel series that are no longer supported out there. Meltdown and Spectre provide more evidence, if it really is needed, that continuing to run old, out-of-date kernels is a terrifically risky thing to do. 

Essentially all of the work that has been merged so far is for the x86 architecture, though others are affected to various degrees. ARM64 is the most prominent of the affected architectures, though it is [only affected](<https://developer.arm.com/support/security-update>) by Meltdown in its high-end processors ([AMD CPUs are not affected](<https://www.amd.com/en/corporate/speculative-execution>) by Meltdown at all). Patches to fix the Meltdown problem on the ARM64 processors where it does exist are [in progress](<https://git.kernel.org/pub/scm/linux/kernel/git/arm64/linux.git/log/?h=kpti>), but will not be merged until the 4.16 merge window. That means they are not available to be added to stable trees currently, so Kroah-Hartman suggests the [Common Android kernel tree](<https://android.googlesource.com/kernel/common/>), which has branches that incorporate the fixes. It is likely that the 4.4 and 4.9 stable trees will never get those fixes, he said: 

For the 4.4 and 4.9 LTS kernels, odds are these patches will never get merged into them, due to the large number of prerequisite patches required. All of those prerequisite patches have been long merged and tested in the android-common kernels, so I think it is a better idea to just rely on those kernel branches instead of the LTS release for ARM systems at this point in time. 

According to Kroah-Hartman, other operating systems (e.g. Windows, macOS) do not yet have full Spectre fixes either. Other architectures have not had fixes for either Meltdown or Spectre, though some are believed vulnerable. The name "Spectre" was chosen, in part, because the researchers believed ""it will haunt us for quite some time"". That looks prescient. 

#### Lessons learned?

There has been additional speculation that other, related problems are either known or will be discovered before long. Regardless, it seems well-nigh certain that CPU security bugs of some kind will be found, so it behooves the industry to try to figure out a less chaotic, and more successful, strategy for handling these kinds of problems. 

Much of the ire has been directed toward Intel, which seems to have taken the lead on the response to these bugs. In [response](<https://lwn.net/Articles/743712/>) to one of the first Spectre-fix postings, Linus Torvalds was characteristically blunt: 

I think somebody inside of Intel needs to really take a long hard look at their CPU's, and actually admit that they have issues instead of writing PR blurbs that say that everything works as designed. 

.. and that really means that all these mitigation patches should be written with "not all CPU's are crap" in mind. 

Or is Intel basically saying "we are committed to selling you shit forever and ever, and never fixing anything"? 

Because if that's the case, maybe we should start looking towards the ARM64 people more. 

Thomas Gleixner, who was singled out for praise by Torvalds for his work in getting the KPTI patches into shape for merging, also [did not mince words](<https://lwn.net/Articles/743713/>). The Spectre bugs have been "fixed" in some distribution kernels, but Gleixner was not impressed with what was done there and is concerned that proper solutions are not being thought out in the name of expediency, which will have long-term implications: 

We have neither the basic mitigations in place nor has anyone fully understood the implications and possible further issues. 

[...] 

I've seen the insanities which were crammed into the distro kernels, which have sysctls and whatever, but at the same time these kernels shipped in a haste do not even boot on a specific class of machines. Great engineering work. 

He decried the ""disgusting big corporate games"" that required him and others to go into panic mode over the past few months, when the chip vendors knew about the problems _months_ before the kernel community was engaged (to the extent it was). He noted that "brain" is not an acronym for ""Big Revenue All Intelligence Nuked"" but it certainly appears to him that it is being treated that way. 

In general, the normal kernel review cycle is taking place for the Spectre fixes, but there are some wrinkles. Now that the vulnerabilities are public, there is increased pressure to get some kind of fix, even one with an enormous performance penalty, deployed in the near term. That runs against "normal" kernel thinking; Torvalds and others were [not persuaded](<https://lwn.net/Articles/743717/>) by a "security trumps performance" argument. Few, if any, disagree in the abstract, but Gleixner and others [would like to roll out the expedient fixes](<https://lwn.net/Articles/743718/>); after that, optimizations can be examined: 

The exploits are out in the wild and they are observed already, so we really have to make a decision whether we want to fix that in the most obvious ways even if it hurts performance right now and then take a break from all that hell and sit down and sort the performance mess or whether we want to discuss the right way to fix it for the next 3 month and leave all doors open until the last bit of performance is squeezed out. 

He suggested that a "performance first" attitude might lead him to taking some overdue vacation time. But James Bottomley [said](<https://lwn.net/Articles/743720/>) that this is all part of the normal kernel review process, which generally leads to better and faster solutions. Alan Cox [agreed](<https://lwn.net/Articles/743721/>), but feels the urgency for an immediate solution: ""I'd just prefer most users machines are secure while we have the discussion and while we see what other architectural tricks people can come up with"". 

All of that discussion is pretty normal stuff for the kernel mailing list. The problem is that it is happening after the disclosure of the bugs, so there is a big (and highly public) clamor. Security bugs are frequently handled privately by the team behind security@kernel.org, but that mechanism apparently was not used here. As Kroah-Hartman [outlined](<https://lwn.net/Articles/743729/>), that led to this sad state of affairs: 

I will note that the "controlled disclosure" for this thing was a total and complete mess, and unlike any that I have ever seen in the past. The people involved in running it had no idea how to do it at all, and because of that, it failed miserably, despite being warned about it numerous times by numerous people. 

[...] 

Because that group was so small and isolated that they did not actually talk to anyone who could actually provide input to help deal with the bug. 

So we are stuck now with dealing with this "properly", which is fine, but please don't think that this is an excuse to blame "controlled disclosure". We know how to do that correctly, it did not happen in this case at all because of the people driving the problem refused to do it. 

We will be digging out from Meltdown and Spectre for a long time. With luck, those responsible have learned that there are better ways to handle bugs of this nature (or any other) in the future. It's vanishingly unlikely that we won't hit this situation again—CPUs are extremely complex beasts and will only get more so as nearly anything gets sacrificed on the altar of performance. The CPU vendors—and any who enabled them (e.g. Google, Microsoft, Apple, Amazon, and so on)—should take a hard look at their practices and this incident in particular. Hopefully, they (and we) will learn something from it moving forward. 

We also owe a huge debt of gratitude to all of the different folks who worked on getting us this far. Many of them worked over the holidays when they might have been doing something much more fun than cleaning up this mess. By delaying the involvement of the kernel team—and setting January 9 as the coordinated release date—whoever was driving the disclosure bus did no one any favors. Six months is an enormous window for an embargo, it is somewhat surprising it held up as long as it did. In the end, though, that embargo length may possibly have given attackers a longer run time; it definitely turned the kernel-fixing piece into a fiasco. Some rethinking is clearly in order.
