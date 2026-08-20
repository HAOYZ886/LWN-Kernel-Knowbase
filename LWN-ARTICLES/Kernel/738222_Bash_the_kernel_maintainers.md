---
title: Bash the kernel maintainers
url: https://lwn.net/Articles/738222/
date: "November 6, 2017"
category: Development model
author: "By Jonathan Corbet November 6, 2017 2017 Maintainers Summit"
---

By **Jonathan Corbet**  
November 6, 2017

* * *

[2017 Maintainers Summit](<https://lwn.net/Articles/KernelSummit2017>)

Laurent Pinchart ran a session at the 2017 Embedded Linux Conference Europe entitled "Bash the kernel maintainers"; the idea was to get feedback from developers on their experience working with the kernel community. A few days later, the Maintainers Summit held a free-flowing discussion on the issues that were brought up in that session. Some changes may result from this discussion, but it also showed how hard it can be to change how kernel subsystem maintainers work. 

The first complaint was that there is no consistency in how maintainers respond to patches. Some will acknowledge them right away, others take their time, and others will sometimes ignore patches altogether. James Bottomley defended the last group, saying that it is simply not possible to respond to all of the patches that show up on the mailing lists. The discussions can get nasty or, with some posters, a maintainer can end up stuck in a never-ending circle of questions and reposts. Arnd Bergmann suggested that maintainers could adopt a standard no-reply reply for such situations. 

Laura Abbott said that the low-quality patches tend to be cleanup work, but there are also plenty of developers doing feature work who would appreciate more consistent maintainer responses. Bottomley said that such developers know what they are doing with regard to working with the community already, but Bergmann disagreed. Bottomley went on to say that the best way to get a response to a feature patch is to recruit other users to say that they, too, are interested in that feature. He is not interested in merging features that have a single user. Finding these [![\[Laurent
Pinchart\]](https://static.lwn.net/images/conf/2017/osse/LaurentPinchart-sm.jpg)](<https://lwn.net/Articles/738224/>) users should not be hard, he said; developers working for companies already have a user base to draw from. When he was working at Parallels, he said, he was able to go out and find users to push for patches that had been languishing. 

Shuah Khan said that there is a lot of inconsistency around when submitters should ping maintainers for a response. Bottomley replied that his "rule number one" for SCSI patches is that submitters must find a developer to review their work. Bergmann said that he knows that the SCSI subsystem works that way, but noted that other subsystems have different rules. In each case, the specific rules in force make sense to the maintainers involved. For the core ARM tree, for example, patches have to be added to the maintainer's patch tracker. In the virtual filesystem (VFS) area, patches have to be "really good", at which point they will silently appear in a mainline pull request. Linus Torvalds added that, for VFS patches, the maintainer tends to be more responsive if Torvalds is copied on patches. Patches for networking code always get a timely response. Ben Herrenschmidt said that he often gets responses rejecting patches for "stupid stuff" (trivial issues like coding style) and that he finds it frustrating. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Torvalds said that it will never be possible for all of the kernel's subsystems to be consistent in their handling of patches. Different subsystems have different models of development that have grown over time. The networking subsystem is good at taking patches, he said, because that's simply how the maintainer (Dave Miller) works; the patchwork system also works well for him. "But nobody else should try to be Dave". A better approach, he said, might be to create better documentation of each subsystem's rules. 

Herrenschmidt said that the community likes to complain about companies that are shipping a lot of out-of-tree code. We would like them to get that code into the mainline. But that task is frustrating and demotivating now, so many of these companies decide that it's not worth the effort; the community needs to make it easier. He repeated that some maintainers are overly obnoxious about trivial issues, making it hard to get large work upstream. The fact that the rules vary across subsystems and are not written down makes it worse. 

The group concluded that an effort should be made to document each subsystem's rules for patch acceptance. The information might be added to the [submitting-patches](<http://static.lwn.net/kerneldoc/process/submitting-patches.html>) document, though that document is already far too long. Perhaps, eventually, the `get_maintainer.pl` script could be enhanced with a better understanding of subsystem-specific rules. The documentation is hopefully forthcoming in the near future but, at the session, the documentation maintainer warned that his rules for patch acceptance are especially nasty. 

Moving on, Pinchart noted that the community could make an effort to be nicer to new developers, preferably in a way that doesn't frustrate existing developers. He noted, for example, that the automated emails from the 0-day testing service can seem harsh — the first response to a first-time patch is essentially an automated criticism. Perhaps, he said, the messages could explain that they originate from a robot and should not be taken personally. Dan Williams said that he has arranged for separate 0-day testing just to avoid this sort of situation. 

There was a suggestion to set up a new mailing list for the purpose of running a specific patch through the testing service. One potential problem is that the robot currently generates no reply if it finds no problems with the patch; that would have to change in this setting. Silence in response to patches, Kees Cook noted, is demotivating. Torvalds suggested that 0-day replies could start with "I love you but..."; Fengguang Wu, who was at the Summit, seems to have already [made some changes](<https://lwn.net/Articles/738223/>) in that direction. Pinchart said that he always starts off a review by thanking the submitter for the patch and that he often offers a Reviewed-by tag if specific changes are made. That gives both a positive message and a hint of light at the end of the tunnel that can help new submitters. 

Patch submitters could benefit from better feedback on how to fix problems, especially in cases where reviewers give conflicting advice. In such cases, it was said, the maintainer needs to take a stand to resolve the situation, but Bottomley said that, if reviewers cannot agree on a patch, he doesn't want it at all. It is up to the submitter, he said, to bring the reviewers to some sort of agreement. Herrenschmidt complained that, sometimes, patches are simply bikeshedded to death; Ted Ts'o suggested escalating to Torvalds when that happens. Torvalds added that he will indeed route around obstructive maintainers when he has to. 

There was a brief discussion regarding a current patch logjam in the SCSI target subsystem. The maintainers agreed that they would go ahead and take the more straightforward changes in an attempt to move things forward. 

Sometimes an interesting patch becomes abandoned, often because the developer has moved on to other tasks. Perhaps reviving such patches would be a good project for an Outreachy intern. Bottomley said that one useful signal about the wisdom of accepting a specific patch is whether it is being pushed consistently; that suggests that the developer will be around to deal with problems after it is merged. Torvalds agreed that the abandonment of a patch says something, and that such patches probably should not be applied. 

The problem of developers who have limited time to get a patch merged remains, though. Ts'o said that he will make an effort to respond quickly to developers who are known to be going away soon — interns, for example. Herrenschmidt said that he has seen some maintainers giving rude responses to developers who are working under deadlines. Bottomley responded that such patches are often created by contractors; they tend to be bad and are best left out. But such patches then just languish in some Android vendor tree, which is not a good outcome either. Ts'o said that the community's normal expectation is that the developer of a patch will stay around to maintain it after merging; Herrenschmidt agreed that contractors need to have some sort of convincing maintenance story for their work. 

The topic of aggressive behavior on the mailing lists came up; it was agreed that all developers should call out such behavior when they see it. Bottomley wondered what the problem was, since the mailing lists have been steadily calming down for years. Apparently one remaining problem is perceived aggression in commit messages, where the changelog for a fix will say unflattering things about the patch that introduced the problem in the first place. There was relatively little sympathy in this case, though. 

There is evidently at least one company that will not assign female developers to specific subsystems because of problems that have been experienced in the past. Everybody agreed that such situations should be taken immediately to the Linux Foundation Technical Advisory Board for resolution. 

At the end of the session, Pinchart said that he was thinking about starting some sort of maintainer survey, patterned on the teacher evaluations used at many universities. This effort is likely to proceed, initially as an opt-in mechanism for maintainers who are interested. The feedback provided would be anonymized and would not be made public. 

[Your editor would like to thank the Linux Foundation, LWN's travel sponsor, for supporting his travel to this event].
