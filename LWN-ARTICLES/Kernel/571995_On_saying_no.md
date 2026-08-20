---
title: "On saying \"no\""
url: https://lwn.net/Articles/571995/
date: "October 29, 2013"
category: Development model; O TMPFILE
author: "By Jonathan Corbet October 29, 2013 2013 Kernel Summit"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
October 29, 2013

* * *

[2013 Kernel Summit](<https://lwn.net/Articles/KernelSummit2013/>)

Are we getting too lax in our acceptance of new code in the kernel? Christoph Hellwig ran a session to explore that question. It may be, he said, that kernel maintainers are getting old and just don't have the energy to fight against substandard code like they once used to; there is very little pushback against ideas now. The review process, he said, is now focused on white-space issues rather than on whether the patch is needed at all. Nobody is willing to just decide that a problem is too hard, resulting in things like the "Linux security modules debacle." People don't want to make decisions or stick their necks out to turn stuff down. There are, Christoph said, two types of maintainer currently: those who merge everything, and those who ignore everything. 

Olof Johansson claimed that he and Arnd Bergmann have gotten pretty good at pushing crap out of the ARM tree. There are a lot of new subsystems with energetic new maintainers, which helps. The jury is still out, he said, but things are running well at this point. 

Ted Ts'o asked for specific examples of code that should not have been merged. Christoph responded that, once upon a time, it used to be hard to merge patches that bloated core kernel data structures; now everything just keeps growing. He also pointed to the various filesystem notification subsystems in the kernel, leading to a threat from Linus: he will kick [![\[Christoph Hellwig\]](https://static.lwn.net/images/conf/2013/lce-ks/ChristophHellwig-sm.jpg)](<https://lwn.net/Articles/572023/>) anybody who tries to submit another notification scheme. Linus did allow that we are letting a lot of stuff through; he relies on the top-level maintainers to push back and that isn't always happening, despite the fact that he asks maintainers to say "no" more often. 

Ted said that, ten years, ago, patches went through a lot more review by developers other than the maintainer of the affected subsystem, but we just can't scale to that kind of review anymore. So maintainers are not paying much attention to what is happening outside of their own subsystems, and a lot of people have tuned out of the linux-kernel mailing list entirely. Ingo Molnar suggested that we are seeing some of the natural results of a distributed development model; the social structure, too, is more distributed. It was also pointed out that heavy criticism is frowned upon more than it used to be. 

Christoph went on to mention the [problems with O_TMPFILE](<https://lwn.net/Articles/562294/>), which he described as "a trainwreck." There are still critical bugs being fixed with that code. In general, he said, it often seems that code going into the kernel has been rushed and hasn't had time for proper review or testing. Dave Jones added that, sometimes, maintainers abuse Linus's trust and merge code that really is not ready to go in. 

Is the model of having a single maintainer for a subsystem still appropriate? Christoph noted that having multiple maintainers seems to help to ensure adequate review of patches. Perhaps group maintenance should become more of the rule than the exception. Tejun Heo responded that we just don't have the manpower to have multiple maintainers for a lot of subsystems. He cited the workqueue code which, he said, really needs somebody else with a deep understanding of what is going on there, but, which, for years, was looked at by nobody but him. 

Andrew Morton said that he often struggles with the question of whether patches should be merged at all. He pointed to the "zstuff" — transcendent memory and related in-memory compression technologies; he kept pushing back against the pressure to merge that code, hoping for an explanation of why we needed it. In the end, a couple of distributors picked it up; that tipped the balance and made it hard to keep that code out. Peter Zijlstra described this as a sort of side-channel attack: distributors will carry almost anything if their users bother them enough. So, by harassing distributors enough, developers can get controversial code merged into the kernel. 

What about patches that cause performance regressions? The sense in the room was that things have gotten a little bit better in that area. Some distributors are running more performance tests; this includes Mel Gorman's work at SUSE and the tests run by Martin Petersen at Oracle. Red Hat, too, has been pushed to run more tests, but the people involved apparently feel a little ignored at the moment. 

Andrew came back in to repeat that he could really use more help in deciding which features should or shouldn't be merged. While being wary of the idea that every problem can be solved with a new mailing list, he thought that perhaps some sort of "graybeards@" list might be helpful for this kind of question. Linus agreed, suggesting that it should be a non-optional list and that all Kernel Summit attendees should be subscribed. If anybody complains loudly, they could be removed, and the community would know that they aren't interested in core issues and don't want to attend any more summits. 

There was some agreement on the creation of this list, but it was also agreed that, somehow, the volume would need to be kept down. Linus suggested that it should not allow messages that are copied to any other list. There may also be a rule that no patches are posted to the list; anything posted there would include links to discussions elsewhere. Ted closed out the session by saying that he would go ahead and create the list and the members could work out the rules from there. 

[Next: [Bugzilla, lightning talks, and future summits](<https://lwn.net/Articles/571997/>)].
