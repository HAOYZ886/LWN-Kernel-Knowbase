---
title: Backporting XFS fixes to stable
url: https://lwn.net/Articles/934941/
date: "June 20, 2023"
category: "Development model-Stable tree; Filesystems-XFS"
author: "By Jake Edge June 20, 2023 LSFMM+BPF"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jake Edge**  
June 20, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

Backporting fixes to stable kernels is an ongoing process that, in general, is handled by the stable maintainers or the developers of the fixes. However, due to some unhappiness in the XFS development community with the process of handling stable fixes for that filesystem, a different process has come about for backporting XFS patches to the stable kernels. The three developers doing that work, Leah Rumancik, Amir Goldstein, and Chandan Babu Rajendra, led a plenary session at the [2023 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023>) (with Babu participating remotely) to discuss that process. 

Goldstein began by noting that each of the presenters is responsible for a different stable kernel; he does 5.10, Rumancik handles 5.15, and Babu is responsible for 5.4. The session was meant to be something of a case study, because other filesystems (and subsystems) have similar issues. He was "very happy to see" that stable maintainer Sasha Levin was present for the session so that he could offer his perspective as well. 

#### History

He put up a graph (seen below) of XFS backports to stable kernels from their [slides](<https://docs.google.com/presentation/d/1CMCKT8cRVq7IoDhJYsjd7tFrgVqWdZxlhsprgW7qHWI/edit?usp=sharing>); it depicted the last five years of development with lines showing the cumulative number of backports for each of six different stable kernels. It is roughly a time-series plot, where he tried to align the stable-tree XFS activity, he said. 

> [ ![\[XFS graph\]](https://static.lwn.net/images/2023/xfs-graph-sm.png) ](<https://lwn.net/Articles/935384/>)

"We can see some drama going on here", he said. The graph shows that five years ago there was "an OK period for XFS backports", but around the "0" on the horizontal axis, which corresponds to the release of the 5.10 kernel, the graph flattens out for quite some time. There is also a slowdown around "-100" on the graph for the 4.14 and 4.19 kernels; around that time, there was a clash between XFS maintainers and stable-tree maintainers about the testing that is being done for stable backports. A testing process was established, but it took some time for Levin to implement it; he led a [session about the testing process](<https://lwn.net/Articles/789225/>) at LSFMM 2019\. 

[ ![\[Amir Goldstein\]](https://static.lwn.net/images/2023/lsfmb-goldstein2-sm.png) ](<https://lwn.net/Articles/935385/>)

The long plateau after the 5.10 release was not really caused by any big disputes, Goldstein said, but was the combination of a few different things. There were some [problems with XFS backports](<https://lwn.net/Articles/838819/>) around that time, which he mentioned in [his session on testing and stable trees](<https://lwn.net/Articles/896523/>) at last year's LSFMM+BPF. Levin changed the amount of time that an XFS patch needed to stay in the mainline before it could be pulled into stable, which addressed the complaint, but the real problem was elsewhere. 

Levin essentially could not keep up with maintaining his test infrastructure, dealing with changes in fstests, and running the tests; all of that work was taking too much of his time. He also changed employers around then, which contributed as well. There was a high bar for testing set by the XFS maintainers, but there was no one to do those tests, so for two years no XFS backports were being done, Goldstein said. 

None of the users of the distributions that were using, say, 5.10 were made aware of the fact that the "stable kernel" from their distributor was not getting updates for XFS. "Nobody told them" that XFS, which is a well-maintained filesystem, was languishing, he said. At the summit last year, he, Luis Chamberlain, Rumancik, and others got together to set up the new system with three maintainers who are taking the lead for a particular stable kernel. 

The 5.15 work is being backed by Rumancik's employer, Google, while the 5.4 maintenance is being supported by Oracle, Babu's employer; in both cases, the companies have a business need to support XFS in those stable kernels, Goldstein said. On the other hand, the 5.10 maintenance is coming from the community; he started working on it because his employer, CTERA Networks, had a need for it, but now it is volunteer work on his part. It is also using community resources from Chamberlain's kdevops work and for a machine to run the tests on (which are contributed by Samsung); if that were not the case, he would have the interest in doing the work, but not the ability to do so. 

Prior to the emergence of this new maintenance model, he was doing the backports for his company, but was not able to do the testing needed to get the fixes into the stable releases. There is a need for companies to contribute maintenance help for kernels they care about, but also to contribute resources for community testing. That will allow efforts to maintain "orphan releases like 5.10". There is also a question of who will be picking up XFS maintenance for the relatively new 6.1 LTS kernel. 

#### Ad hoc backports

Rumancik said that there is a problem that backports to stable trees are handled in a fairly ad hoc fashion. Fix authors will sometimes backport the fixes to some or all of the stable branches where the fix is needed, but sometimes they do not do any of that. The Fixes tags and/or copying the stable mailing list on patches can be haphazard; the [AUTOSEL patches](<https://lwn.net/Articles/825536/>), which are [chosen by a machine-learning system](<https://lwn.net/Articles/764647/>), help fill in the gap, but not all of those get applied. 

[ ![\[Leah Rumancik\]](https://static.lwn.net/images/2023/lsfmb-rumancik-sm.png) ](<https://lwn.net/Articles/935386/>)

Patches that do not apply cleanly often get lost because no one follows up on the report. There are patches that might apply but are not even looked at because some critical prerequisite is missing; the fix could still be made to work, but the patch just falls by the wayside. The idea for XFS is that the stable maintainers can keep a closer eye on the patches that might apply; since they are familiar with XFS and that particular stable kernel, they can backport and test the fixes. They generally batch up a few fixes, run them through the testing regimen, then post them to the XFS mailing list; if no shouting is heard within a few days, the patches get sent to the overall stable-kernel maintainers. 

At that point, Darrick Wong came in over the Zoom link to fill attendees in on what Oracle has "been up to with the LTS kernel". Oracle used to operate in the "classic Linux distributor model" by choosing a kernel as the base for its enterprise kernel, then applying random patches and shipping the result to customers. More recently, the company has switched its processes to use the LTS kernels and all of the patches that come in those releases; all of the fixes released for the stable kernel are eventually released in the enterprise kernel "when we get around to doing that". 

Something that he has heard the stable maintainers complain about a little bit is the lack of companies willing to stand up and say that their products are based on the LTS kernels and that they are willing to fund maintenance and backporting activity for those kernels. He reiterated that Oracle does use the LTS kernels; it picks up the odd-year LTS for its enterprise kernel and the company is "totally willing to fund" work on the parts of the kernel that it has experience with and knowledge about, which includes filesystems and storage, Wong said. 

It has gotten to the point where it is easier to get something fixed in the enterprise kernel by getting the fix backported into the upstream stable kernel that it is based on, rather than to go through the internal bug-fixing procedures. Oracle is committed to ensuring that the LTS kernels stay current "for a while"; he has heard rumblings of shorter support windows for the LTS kernels, down from the current six years, but Oracle would like them to not decrease too much. The company recognizes that the stable-kernel effort "does take a considerable amount of engineer time and some amount of cloud resources"; he noted that Oracle is a cloud vendor so it could provide some of those resources as well. 

As the upstream XFS maintainer, Wong said that he is "really really _really_ grateful" to the three stable maintainers for taking on that task. He can just barely keep up with the mainline kernel; in fact, he said that he was a contributing factor to the two-year flatline in the graph due to him not scaling and not keeping up around that time. There have been some internal discussions at Oracle about whether it makes sense to continue to cherry-pick patches for the enterprise kernels, as is done now, or if it would make more sense to "forklift entire releases" of XFS into older kernels. The standard "LOL folios" answer, which refers to the changes for folios that have gone into more recent versions of XFS, makes it seem too difficult to update XFS that way. 

#### Other filesystems

Matthew Wilcox said that he is up for maintaining a folio compatibility layer for older kernels if there is a need for it. It could benefit more than just XFS; if ext4 or Btrfs wants to port newer versions to older kernels, those developers should be talking with Wilcox, he said. James Bottomley said that it is not just for forklift ports, either; the folio changes are invasive enough that regular fixes will be harder to backport in the future. Without some kind of compatibility layer, patches that apply on a folio kernel will not apply on an earlier non-folio kernel; the diffs will simply not really match up. 

Ted Ts'o said that is one of the reasons he would like to recruit stable backport maintainers for ext4 like the team for XFS. He thinks it would be a good way for more junior developers to get more involved in kernel development, beyond simply trying to fix syzbot bugs, for example. If there are companies that want to get their employees up to speed on ext4, backporting fixes to the stable kernels provides a structured way to start out. It is a great service to the community and is less open-ended than diagnosing a syzbot crash; someone has already fixed the bug, so backporting is a matter of transplanting that fix. 

Beyond that, there have been more problems with stable backports for ext4 of late, so he is coming around to the view of the XFS maintainers. There is also the problem that sometimes he is just swamped, such that critical bug fixes fall on the floor due to his lack of bandwidth to look into them. The stable maintainers dutifully inform him that a patch does not apply, but sometimes he has no chance to look at it. Users who are depending on the stable kernels for security fixes may not be getting what they think they are. 

While the summit may not be a great place to recruit for the ext4 stable backports team, he thought attendees might know of others who are interested in learning about filesystems; that kind of work would be a great way to do so. Rumancik said that it is "a bite-sized way to learn about things because you get sets of patches and can just dig into that area", so it is not too overwhelming. You can also watch the corresponding patches that go into other, more recent stable kernels, which helps as well. 

There are some areas that still need work, Rumancik continued, including making it easier to identify stable-backport candidates; she knows that there is resistance to copying the stable mailing list, but it can definitely help alert the maintainers. It would also be good if a standard test procedure could be developed and adopted; right now there are different ideas of how many fstests runs and how many different configurations need to be tested before acceptance. 

Chamberlain said that it might help to be able to see what patches the AUTOSEL tool would have chosen for XFS. Those patches are not being automatically picked up and used, because of the requirements from the XFS maintainers, but they could be consulted as a source of patches that should be considered for backporting. The XFS stable maintainers could review that output if it were available. 

Rumancik said she would be interested in looking at that. Levin said that it had already been implemented for KVM and parts of the x86 subsystem code; patches are sent with a MANUALSEL tag, instead. He has noticed that the number of such patches has drastically reduced over the last few months, perhaps because those subsystems are getting better at tagging their own patches due to the MANUALSEL patches. So the infrastructure already exists to do this, Levin said. 

Chamberlain asked if the infrastructure could be reproduced elsewhere for experiments and the like, but Levin cautioned that "AUTOSEL is a massive pile of tech debt". It is running on an old Azure VM, for example. Chamberlain and Levin plan to work together to make the infrastructure more widely available. 

Bottomley said that there was still an "elephant in the room"; Wong had put up his hand to say that Oracle will assist in the LTS efforts, but none of the other distributions, some of which were represented at the summit, had followed suit. These other distributions have large teams of people backporting fixes; pooling those resources would be beneficial. 

Goldstein said that over the last five years, more of the enterprise distributions have started using the LTS kernels. Both Oracle and SUSE have switched, he said, leaving just Red Hat as the only enterprise distribution that is not based on LTS kernels. But Jan Kara pointed out that SUSE is still using the (non-LTS) 5.14 kernel and it does a lot of backports to that kernel. Those backports may have value for other kernels, such as 5.15 or 5.10, though. Michal Hocko said that the SUSE kernel trees are available for those who want to see which backports have been done, along with the details of how they were done. 

The session was over time at that point, so Rumancik quickly went through some benefits to the approach taken for XFS, which could be applied to other filesystems, such as ext4. There are some efficiencies that come from batching up the changes and testing them together; in addition, working with the other team members and their backports to other branches makes the process easier. Wong closed the session by noting that the Fixes tags greatly help the process of finding patches to backport, but another way to draw attention to a fix is by adding a regression test to fstests for the problem—with a pointer to the patches of interest.
