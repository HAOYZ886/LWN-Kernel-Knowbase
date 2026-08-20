---
title: "XFS, stable kernels, and -rc releases"
url: https://lwn.net/Articles/838819/
date: "December 3, 2020"
category: "Development model-Stable tree; Filesystems-XFS"
author: "By Jonathan Corbet December 3, 2020"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
December 3, 2020

Ever since the stable-update process was created, there have been questions about which patches are suitable for inclusion in those updates; usually, these discussions are driven by people who think that the criteria should be more restrictive. A regression in the XFS filesystem that found its way into the [5.9.9](<https://lwn.net/ml/linux-kernel/1605788188121239@kroah.com/>) stable update briefly rekindled this discussion. In one sense, there was little new ground covered in this iteration, but there was an interesting point raised about the relationship between stable updates and the mainline kernel -rc releases. 

In the beginning, stable updates were restricted to critical fixes only, but the rules were relaxed over time. The patches merged for stable updates now are often [automatically selected](<https://lwn.net/Articles/764647/>) using a machine-learning system; others are picked because they look like they fix something somewhere. The result has been a massive increase in the number of patches going into the stable updates; the 5.9.x series has had over 1,900 patches applied through 5.9.11, while the delta between 4.9 and 4.9.246 is well over 18,000 patches. 

Incorporating all those patches undoubtedly has the effect of increasing the number of useful fixes in the stable releases, which is a good thing. But it also increases the chances of merging bad patches that provide users with something other than the problem-free experience they were looking for. 

For example, [this XFS "fix"](<https://lwn.net/ml/linux-xfs/160494584816.772693.2490433010759557816.stgit@magnolia/>) was posted to the linux-xfs list on November 9; it was reviewed, applied, and eventually [pushed to the mainline](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=6ff646b2ceb0>) four days later, where it appeared in the [5.10-rc4](<https://lwn.net/ml/linux-kernel/CAHk-=wjFfAktnadOPb_iV5nKh=V5Am1sG-gciYveswRtuEkrLQ@mail.gmail.com/>) release. On the 17th, Greg Kroah-Hartman included this patch in the [5.9.9 review cycle](<https://lwn.net/ml/linux-kernel/20201117122138.925150709@linuxfoundation.org/>), along with 254 other fixes. No objections were raised, and the patch was part of the 5.9.9 release on the 19th, ten days after it was originally posted. 

It took three days for Nick Alcock to [report](<https://lwn.net/ml/linux-xfs/87lfetme3f.fsf@esperi.org.uk/>) that he was getting XFS corruption errors when running a current stable kernel. Wong's patch was the source of this problem — indeed, the XFS developers had already [reverted it](<https://git.kernel.org/linus/eb8409071a1d4>) in the mainline for 5.10-rc5 (with Eric Sandeen credited for the report). That revert found its way into the [5.9.11](<https://lwn.net/ml/linux-kernel/160623671354173@kroah.com/>) update on November 24\. So there was a period of about five days when the current stable 5.9 kernel had a bug that made XFS filesystems unusable in some configurations. It's worth noting that, even though XFS reported corruption, no actual filesystems were corrupted; they simply refused to mount. 

That was the end of this incident — at least, until the stable-kernel machine-learning system [picked another XFS patch](<https://lwn.net/ml/linux-kernel/20201125153550.810101-33-sashal@kernel.org/>) for a future 5.9.x release. At that point, Dave Chinner [objected](<https://lwn.net/ml/linux-kernel/20201125215247.GD2842436@dread.disaster.area/>): 

We've already had one XFS upstream kernel regression in this -rc cycle propagated to the stable kernels in 5.9.9 because the stable process picked up a bunch of random XFS fixes within hours of them being merged by Linus. One of those commits was a result of a thinko, and despite the fact we found it and reverted it within a few days, users of stable kernels have been exposed to it for a couple of weeks. That *should never have happened*. 

Chinner said that the stable process is cutting out some of the time that is needed to find problems in patches merged into the mainline, and requested that XFS patches not be selected for stable updates until after they have appeared in a final mainline release. 

Sasha Levin, who maintains the automated patch-selection system, [replied](<https://lwn.net/ml/linux-kernel/20201125234654.GN643756@sasha-vm/>) that the real failing was in the XFS quality-control process which, he said, should never have let the buggy patch through in the first place. By the time something is in the mainline, he said, it is assumed to be correct: 

The stable process assumes that commits that ended up upstream were reviewed and tested; the stable process doesn't offer much in the way of in-depth review of specific patches but mostly focuses on testing the product of backporting hundreds of patches into each stable branch. 

Chinner [took strong exception](<https://lwn.net/ml/linux-kernel/20201126071323.GF2842436@dread.disaster.area/>) to Levin's attempt to pin the blame on the XFS side. The bug, it seems, only manifests in settings where a newer user space is interacting with an older kernel, which is not a scenario that is tested prior to merging patches. Follow-on testing done by the XFS developers did eventually turn up the problem, at which point it was fixed. It is not reasonable, he said, to expect that every patch go through that degree of testing before being applied. 

The important point, he said, was that not all problems can realistically be found before patches land in the mainline: 

I'll repeat the lesson to be learned here: merging a commit into Linus's tree does not mean it is fully tested and production ready. It just means it's passed a wide range of unit tests without regressions and so is considered ready for wider integration testing by a larger population of developers and testers. 

He said, again, that the stable process needs to slow down some to allow for that wider testing to take place, and later [repeated this argument](<https://lwn.net/ml/linux-kernel/20201202204045.GM2842436@dread.disaster.area/>) in a separate discussion, drawing [an objection](<https://lwn.net/ml/linux-kernel/X8gBUc0fkdh6KK01@kroah.com/>) from Kroah-Hartman: 

That means that the world would go for 3 months without some known fixes being applied to the tree? That's not acceptable to me, as I started doing this because it was needed to be done, not just because I wanted to do more work... 

Chinner then [clarified](<https://lwn.net/ml/linux-kernel/20201203015003.GN2842436@dread.disaster.area/>) that most subsystems, as he saw it, did not need this sort of delay. He also suggested that perhaps waiting for two -rc cycles would be sufficient for subsystems where the risk of catastrophic regressions (filesystems, for example) is high. 

The need for more time to test patches before applying them to the stable updates has been recognized in the past. Initially, any patch that had made it into the mainline was considered fair game for stable-update consideration. Later on, though, the rule was changed to require patches to appear in at least one -rc release first. But that still does not give a whole lot of time for problems in the mainline to come to light. 

The fact that -rc releases may contain bugs is implicit in the fact that every development cycle involves seven or eight such releases. Getting into an -rc release is not deemed enough for a patch to appear in a final mainline release; patches are expected to survive testing for a few -rc releases first. But the stable updates can ship those patches immediately after they appear in an -rc release. Thus, in a sense, the mainline, which is not considered stable enough for most installations, has a more stringent requirement than the stable releases, which are widely used. 

The conversations ended at this point, with no indications that any minds were changed. There can be no doubt that this topic will return, though. Bugs introduced into stable releases go against the reasons for using those releases in the first place, so they can be expected to draw attention. The number of such bugs will never be zero, though, regardless of how slow or careful the process is. A balance has to be found between the need to get as many fixes as possible to users as quickly as possible and the need to avoid regressions. If the "after one -rc release" rule allows too many regressions, then perhaps it is time for the community to more clearly articulate what it means when a patch lands in the mainline and adjust the stable-update selection process accordingly.
