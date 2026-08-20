---
title: Intentionally buggy commits for fame—and papers
url: https://lwn.net/Articles/853717/
date: "April 21, 2021"
category: Security
author: "By Jake Edge April 21, 2021"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jake Edge**  
April 21, 2021

A buggy [patch](<https://lwn.net/ml/linux-kernel/20210407001658.2208535-1-pakki001@umn.edu/>) posted to the linux-kernel mailing list in early April was apparently the last straw for Greg Kroah-Hartman as it led to the planned reversion of a whole slew of commits with one thing in common: their origin at the University of Minnesota (UMN). The patch to the NFSv4 authorization mechanism was duly questioned by two NFS developers, but it is not an honest mistake; according to Kroah-Hartman, there has been an attack of sorts underway as part of some academic research at the university. In order to be sure that these intentional bugs, many with security implications, do not continue to haunt Linux, he is working on reverting commits that came from email addresses with the umn.edu domain. 

The buggy patch was posted by Aditya Pakki on April 6, [questioned](<https://lwn.net/ml/linux-kernel/20210407153458.GA28924@fieldses.org/>) by J. Bruce Fields the next day, and [NACKed](<https://lwn.net/ml/linux-kernel/c0de0985c0bf09a96efc538da2146f86e6fa7037.camel@hammerspace.com/>) by Trond Myklebust the day after that. On April 20, Kroah-Hartman [responded](<https://lwn.net/ml/linux-kernel/YH5%2Fi7OvsjSmqADv@kroah.com/>) to it in decidedly unhappy fashion: 

> Please stop submitting known-invalid patches. Your professor is playing around with the review process in order to achieve a paper in some strange and bizarre way. 
> 
> This is not ok, it is wasting our time, and we will have to report this, AGAIN, to your university... 

Fields [asked](<https://lwn.net/ml/linux-kernel/20210420171008.GB4017@fieldses.org/>) for more details, some of which [were filled in](<https://lwn.net/ml/linux-kernel/YH+zwQgBBGUJdiVK@unreal/>) by Leon Romanovsky. A [paper [PDF]](<https://github.com/QiushiWu/QiushiWu.github.io/blob/main/papers/OpenSourceInsecurity.pdf>) by Qiushi Wu and Kangjie Lu, both of the University of Minnesota, details the process of introducing use-after-free bugs into the kernel for the purposes of, essentially, showing that it can be done—and [presenting a paper](<https://www.ieee-security.org/TC/SP2021/program-papers.html>) about it, naturally. Romanovsky continued: ""Yesterday, I took a look on 4 accepted patches from Aditya [Pakki] and 3 of them added various severity security 'holes'."" 

Kernel developers have enough problems with bugs being added by mistake, so patches with intentional bugs are obviously unwelcome. Kroah-Hartman [said](<https://lwn.net/ml/linux-kernel/YH+7ZydHv4+Y1hlx@kroah.com/>) that all of the patches coming from these developers need to be reverted because ""what they are doing is intentional malicious behavior and is not acceptable and totally unethical"". He put together a [patch set](<https://lwn.net/ml/linux-kernel/20210421130105.1226686-1-gregkh@linuxfoundation.org/>) of 190 reversions that he called the "easy" reverts; there is a set of 68 additional patches that need manual review to determine what to do about them. ""Some of them are not able to be reverted as they already have been reverted, or fixed up with follow-on patches as they were determined to be invalid. Proof that these submissions were almost universally wrong."" 

He asked that maintainers review the 190 reversions to see if some actually fixed real problems; so far, a small handful have been determined to either be [real fixes](<https://lwn.net/ml/linux-kernel/20210421092919.2576ce8d@gandalf.local.home/>) or to [be harmless](<https://lwn.net/ml/linux-kernel/9c52814a-7db1-1989-6fd2-e6d80f9a92ad@suse.com/>). The real fixes, at least, will remain in the tree. Kroah-Hartman had a warning for maintainers, though, in light of this situation: 

> [...] but [maintainers] should be aware that future submissions from anyone with a umn.edu address should be by default-rejected unless otherwise determined to actually be a valid fix (i.e. they provide proof and you can verify it, but really, why waste your time doing that extra work?) 

Of course, it is not just the mainline that is affected by these dubious patches. As Sudip Mukherjee [pointed out](<https://lwn.net/ml/linux-kernel/CADVatmNgU7t-Co84tSS6VW=3NcPu=17qyVyEEtVMVR_g51Ma6Q@mail.gmail.com/>), some have already made their way into stable kernels. Those [need reversion](<https://lwn.net/ml/linux-kernel/YH%2F8jcoC1ffuksrf@kroah.com/>) too, Kroah-Hartman said. 

There is a question of whether this type of research is ethical. Abhi Shelat [said](<https://lwn.net/ml/linux-kernel/3B9A54F7-6A61-4A34-9EAC-95332709BAE7@northeastern.edu/>): ""Academic research should NOT waste the time of a community."" He suggested reporting the behavior to the review board that is responsible for ensuring that research at UMN remains ethical and, in particular, anything that might be considered [human experimentation](<https://en.wikipedia.org/wiki/Human_subject_research>) is supposed to be pre-reviewed by that board. While making a report of that nature may happen, kernel developers have a much more direct "fix", as Romanovsky [pointed out](<https://lwn.net/ml/linux-kernel/YIAYThdIoAPu2h7b@unreal/>): ""Our solution to ignore all @umn.edu contributions is much more reliable to us who are suffering from these researchers."" 

That fix is, of course, fairly drastic; there are surely some folks with umn.edu addresses that are not part of the research, nor trying to intentionally slip bugs into the kernel. In fact, Pakki claimed to not be purposely doing so in email to Kroah-Hartman, who [replied](<https://lwn.net/ml/linux-kernel/YH%2FfM%2FTsbmcZzwnX@kroah.com/>) on the list. Pakki said that the patches were based on the output of a new static analyzer that he wrote, but its ""sensitivity is obviously not great"". Kroah-Hartman was not buying that explanation, however: 

> Our community welcomes developers who wish to help and enhance Linux. That is NOT what you are attempting to do here, so please do not try to frame it that way. 
> 
> Our community does not appreciate being experimented on, and being "tested" by submitting known patches that are either do nothing on purpose, or introduce bugs on purpose. If you wish to do work like this, I suggest you find a different community to run your experiments on, you are not welcome here. 

Of course, it is hard to imagine that other communities are looking forward to having their time wasted with known-buggy patches either. In fact, a thorough review of patches emanating from UMN over the last year or two is probably in order for other projects, particularly high-profile ones. There is, of course, nothing stopping "researchers" from using other kinds of email addresses for these patches, though the umn.edu domain does at least provide a fig leaf of cover; otherwise, introducing security holes into the kernel from a personal address is pretty much indistinguishable from an attempt to backdoor the kernel—which may be a criminal offense. In addition, a maintainer receiving email from a .edu domain may be less likely to deeply scrutinize a patch that looks like a simple bug fix. 

It may not quite have sunk in yet, but these researchers have damaged not only their own reputations, but the entire university's, which is obviously unfair, but there are not really any workable alternatives. Some efforts at damage control have been tried, but the statements made do not mesh well with the situation on the ground. On the [home page](<https://www-users.cs.umn.edu/~kjlu/>) for Lu, one of the authors of the paper, there is a note and a link to a three-page [clarification [PDF]](<https://www-users.cs.umn.edu/~kjlu/papers/clarifications-hc.pdf>) on the entry for the paper. The note says: 

> The experiment did not introduce any bug or bug-introducing commit into OSS. It demonstrated weaknesses in the patching process in a safe way. No user was affected, and IRB exempt was issued. The experiment actually fixed three real bugs. 

The clarification continued in the same vein, claiming that no code used by the study made it into the kernel; it did note that ""unfortunately"" maintainers' time was wasted in the process, however. But various kernels have been found with bugs introduced by suspect commits. In fact, Guenter Roeck, in a reply to an apparently private email from Lu [pointed to](<https://lwn.net/ml/linux-kernel/20210421154949.GA168854@roeck-us.net/>) a [commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=c9c63915519b>) authored by Lu that introduced a problem. Roeck probably summed up the thinking of many in his reply: 

> It might be worthwhile to have a discussion at the upcoming maintainers summit on how to handle contributions from untrusted sources in the future, and how to identify trusted contributors. Quite obviously the paradigm has finally changed from "assume the contribution is well-intended" to "assume the contribution is malicious". I guess that was prone to happen, but it is sad to experience it anyway. 
> 
> For my part, congratulations (in a negative sense): You made me much less inclined to accept minor bug fixes from people I don't know in the future. 

UMN is now aware of the problems; the Computer Science department [put out a statement](<https://cse.umn.edu/cs/statement-cse-linux-kernel-research-april-21-2021>) acknowledging the concerns expressed by the kernel community. An investigation will be undertaken: 

> We take this situation extremely seriously. We have immediately suspended this line of research. We will investigate the research method and the process by which this research method was approved, determine appropriate remedial action, and safeguard against future issues, if needed. We will report our findings back to the community as soon as practical. 

It is a horrifically messy situation, seemingly brought about by researchers who were not too concerned about the effects of their research on others. While Linux developers hardly need the additional work, the resulting "extra" scrutiny of new patches will be beneficial. It is a bit hard to see that as a silver lining, exactly—more like a sad but necessary outcome that was inevitable, as Roeck put it. This incident also should serve as a warning to researchers, at least hopefully, going forward: our communities are not playthings. Our code is free and open, but not for abuse.
