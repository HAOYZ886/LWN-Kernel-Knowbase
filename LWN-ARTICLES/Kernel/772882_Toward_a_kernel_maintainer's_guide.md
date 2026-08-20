---
title: "Toward a kernel maintainer's guide"
url: https://lwn.net/Articles/772882/
date: "November 26, 2018"
category: Documentation
author: "By Jonathan Corbet November 26, 2018 Linux
Plumbers Conference"
---

By **Jonathan Corbet**  
November 26, 2018

* * *

[Linux Plumbers Conference](<https://lwn.net/Archives/ConferenceByYear/#2018-Linux_Plumbers_Conference>)

"Who's on Team Xmas Tree?" asked Dan Williams at the beginning of his talk in the Kernel Summit track of the 2018 [Linux Plumbers Conference](<https://linuxplumbersconf.org/>). He was referring to a rule for the ordering of local variable declarations within functions that is enforced by a minority of kernel subsystem maintainers — one of many examples of "local customs" that can surprise developers when they submit patches to subsystems where they are not accustomed to working. Documenting these varying practices is a small part of Williams's project to create a kernel maintainer's manual, but it seems to be where the effort is likely to start. 

In theory, Williams said, kernel maintenance is a straightforward task. All it takes is accumulating patches and sending a pull request or two to Linus Torvalds during the merge window. In this ideal world, subsystems are the same and there is plenty of backup to provide continuity when a maintainer takes a vacation. In the real world, though, the merge window is a stressful time for maintainers. It involves a lot of work juggling topic branches, a lot of talking to people (which is an annoying distraction), and the fact that Torvalds can instinctively smell a patch that is not yet fully cooked. Maintenance practices vary between subsystems, and there is no backup for the maintainers in many of them. It is hard for a maintainer to take a break. 

Kernel maintainers, he said, are a gang of opinionated people. They don't always agree on things, but the good news is they don't have to. So why would we want a maintainer's handbook? The idea is to create a reference [![\[Dan Williams\]](https://static.lwn.net/images/conf/2018/lpc/DanWilliams-sm.jpg)](<https://lwn.net/Articles/772883/>) manual for both maintainers and contributors, a collection of "tribal knowledge" and best practices rather than a set of rules. There is a lot of good advice for maintainers to be found in email discussions, but nobody has, yet, gone to the effort to capture that information and present it in a useful form. 

Another way of putting it, he said, is that there is a fair amount of pain in the community, and he would like to try to alleviate it. Contributors feel the pain of trying to get a maintainer to do something; maintainers, instead, feel the pain of simply trying to hold everything together. He noted that he, too, is guilty of doing things as a maintainer that have caused him stress as a contributor; it is easy to unintentionally make the process harder for others. By addressing some of those pain points, Williams hopes he can help to create a better experience for all involved. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

For example, one painful experience for contributors is getting silence in response to patches sent to a maintainer. Different maintainers exhibit different latencies, so it is hard to know when to press further. One way to address this problem might be for maintainers to advertise an equivalent to a service-level agreement (SLA) documenting the response time they agree to provide. Associated with the SLA could be information like a set of trusted reviewers who could stand in for the maintainer for many review tasks, the location of the subsystem's test suite, and more. By setting the contributor's expectations, the guide should make their life easier; they will know when to resend a patch. 

Another part of the guide would concern itself with preventable maintainer mistakes. There have been a lot of lectures on the proper use of Git posted by Torvalds (and others) over the years; it should be collected and put into a place where maintainers can find it before they make a mistake. Torvalds, Williams said, provides great explanations of how things should be done "after the storm passes"; he does so patiently, repeatedly, as the same mistakes are discovered anew. Why, Williams asked, isn't this information written down anywhere? 

As Williams looked into the creation of a kernel maintainer's guide, he discovered that [one already exists](<https://www.kernel.org/doc/html/v4.19/maintainer/index.html>); it was created by Tobin Harding in 2017 and hasn't been changed since. His first objective is to [add subsystem profiles](<https://lwn.net/ml/linux-kernel/154225759358.2499188.15268218778137905050.stgit@dwillia2-desk3.amr.corp.intel.com/>) to this guide; the profile is meant to tell contributors how to work with the subsystem. It would include information like: 

  * Whether the subsystem accepts pull requests or, instead, requires that all submissions be posted as patches to a mailing list. 
  * The last day before the merge window that new features can be posted and the last day that any new features could actually be merged. This "last day" is likely to be expressed in terms like "when -rc5 comes out". 
  * What the requirements are for Reviewed-by or Acked-by tags on patches and whether the maintainer is allowed to merge unreviewed patches. 
  * Whether the subsystem has a test suite and where it can be found. 
  * A list of trusted reviewers for the subsystem. 
  * The "resend cadence" for the subsystem — how long should a contributor wait before resending a patch? 
  * The time zone(s) in which the maintainers operate, which would be a hint for when contributors could expect a response to an email. 
  * The maintainer's opinion on trivial cleanup patches. 
  * Whether the maintainer trusts off-list patch reviews. These often take the form of a Reviewed-by tag from somebody who works for the same company as the submitter; not all maintainers put much faith in such tags. 

If nothing else, this list is an interesting overview of how different kernel maintainers approach their job. It drew the obvious question ("why do we have these differences in the first place?") from the audience, but there isn't really an answer beyond "it has always been that way". 

The session wound down with some unfocused discussion on the details of the subsystem profiles. Should they be listed in the MAINTAINERS file? Should they include the maintainer's expectations on the documentation of new features? Answers to those questions will have to await the conclusion of the wider discussion, which is ongoing as of this writing. But, as Mel Gorman noted at the end of the talk, the work that has been done so far is a useful enumeration of the problem space, which is a good start. 

[Thanks to the Linux Foundation, LWN's travel sponsor, for supporting my travel to the event.]
