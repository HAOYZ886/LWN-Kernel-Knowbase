---
title: The uncertain future of kernel regression tracking
url: https://lwn.net/Articles/990599/
date: "September 19, 2024"
category: "Development model-Regressions"
author: "By Jonathan Corbet September 19, 2024 Maintainers Summit"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
September 19, 2024

* * *

[Maintainers Summit](<https://lwn.net/Articles/990740/>)

Tracking of regressions seems like an important task for any project; there is no other way to ensure that known problems are fixed. At the 2024 Maintainers Summit, though, Thorsten Leemhuis, who has been doing that work for the kernel, expressed some doubts about whether it is worth continuing. The result was an energetic session on how regression tracking should be done better, and how this work should be supported. 

Leemhuis began by saying that he is thinking about giving up on regression tracking. The funding that was supporting this work has gone away. On top of that, this work has resulted in a number of "annoying" discussions with maintainers who do not appreciate being nagged about open regressions. He does not really even know what Linus Torvalds expects with regard to regression tracking and fixes. Burnout is a problem for many maintainers, and being pressed to fix regressions can make it worse, but burnout is a problem for Leemhuis as well 

[![\[Thorsten
Leemhuis\]](https://static.lwn.net/images/conf/2024/ms/ThorstenLeemhuis-sm.png)](<https://lwn.net/Articles/990739/>) He made a request for some basic guidance regarding the expectations in this area. His reports to Torvalds on open regressions often get no reply at all; Torvalds answered that he tends not to answer email unless it is truly necessary. 

A lack of support from developers and maintainers is making the regression-tracking task harder. There is [a mailing list for discussions on regressions](<https://lore.kernel.org/regressions/>), but almost nobody adds it to the CC list. He understands why; few people want to feel that he is watching them, but including that list would help a lot. The use of `Closes` tags on patches that fix regressions would also be useful. 

Leemhuis said that he often gets responses along the lines of "you are not my manager" when he reminds maintainers of regressions in their subsystem. It makes him feel bad; "nobody likes cops" and he is not enjoying being one. 

With regard to tools, dealing with [the kernel bugzilla](<https://bugzilla.kernel.org/>) remains annoying. Additionally, he cannot add bugzilla participants to email CC lists due to GDPR rules; email addresses provided there are personally identifying information, and the site's policy does not allow exposing them in that way. Some subsystems, he said, are using him as a go-between with bugzilla, but he is not paid to do that work. 

His [`regzbot`](<https://gitlab.com/knurd42/regzbot>) regression-tracking tool works, he said, but he has never been a good programmer and would like some help with it. He has, still, added GitHub support to regzbot and can track regressions reported there. In the absence of funding, though, he is unlikely to continue this work. 

He mentioned the possibility of maintaining a separate Git tree for regression fixes that could be sent upstream if subsystem maintainers do not send fixes themselves. Torvalds said that he has pulled that sort of fixes branch in the past but did not like it; that is not the right way to bypass maintainers. But perhaps it is necessary, he said, when regressions are not being fixed. Jens Axboe suggested that this sort of bypass should be normal practice for any regressions that are still present when the development cycle reaches -rc6. 

#### Fix or revert

The discussion had mostly focused on reverting changes that cause regressions, but Torvalds pointed out that there is often a known fix for any given bug, so a revert is not always the right approach. Fixes are better than reverts, but he does not want anybody to rely on third parties getting fixes upstream. Will Deacon said that maintainers, in general, strongly prefer to not have regressions in their subsystems and they do not just ignore them. It can take a while to get a fix in, though; fixes need testing and integration work. It can be annoying to be nagged by Leemhuis while this work is ongoing. 

Leemhuis responded that he does not normally bother maintainers if he is able to see any sort of activity aimed at a problem. He does get a bit more aggressive if a regression has found its way into the stable trees, though; in that case, the fix cannot be backported until it hits the mainline, so there is an additional reason to hurry. Torvalds said that his normal deadline for fixes is -rc6; if nothing has arrived by then he will consider other actions. 

Ted Ts'o said that there can be a few failure points in this whole process. One is when maintainers acknowledge a regression, but hold off on applying a fix. A different sort of failure happens in the case of sporadic maintainers, who may not be paying attention at the time at all. He once fell down on a regression that he had not realized had found its way into a released kernel. Since he did not know that real users were affected, he did not prioritize a quick fix; better reporting would have helped in that case, he said. 

Ts'o also noted that this is the first he has heard that the regression-tracking work is no longer funded; fixing that should be "a no-brainer", he said. Thomas Gleixner added that the kernel needs to apply some sort of "sustainability tax" so that this work (and more, including addressing technical debt) can be supported. Ts'o said that developers should push their employers to support cleanup work, but Gleixner was pessimistic about that approach; proper funding and dedicated people are what is needed. 

Kees Cook asked Leemhuis whether he wants to continue this work if funding can be found; if so, what other changes would he like to see? Leemhuis said that he would like to continue, but that he wants to see some agreed-upon guidelines about regression response and what his role should be. Deacon said that there are two jobs involved: tracking regressions (which everybody wants more of), and chasing developers for fixes, which is less popular. Perhaps, he suggested, the responsibilities should be separated. 

Rafael Wysocki said that, when he did regression tracking years ago, he only did the tracking part. Torvalds said that he likes it when somebody else is the bad guy for once. But, he said, adding more people to the task is not going to improve fix times. He suggested working more on automated tracking and nagging; people respond less emotionally to an automated message. But, he said, it would also be good to just impose a policy saying that, after a given number of weeks without a fix, a patch causing a regression will simply be reverted. 

#### Guidelines

With regard to guidelines, Ts'o suggested that, normally, a fix should land in linux-next within a week of a regression report. Jason Gunthorpe said that, sometimes, a regression has wide-ranging impact, causing automated testing to fail. Those regressions need to be fixed quickly; that can involve escalating a situation to Torvalds, which nobody likes. Torvalds, though, said that escalation is the right thing to do; he is happy to aggressively revert changes that break testing. Having a patch reverted that way does not need to carry a stigma; it is simply something that happens at times. 

Dan Williams suggested that a fully automatic process for reverting buggy commits would help to reduce the stigma associated with reversion; Torvalds agreed, saying that would make things less personal. Deacon suggested putting the responsibility on maintainers to do those reverts; Torvalds said he would like that, but he feared that each additional person in the chain would add latency before a fix gets to the mainline. Alexei Starovoitov said that, in some subsystems, reverts are just standard procedure; Torvalds worried that such a policy could encourage people to apply non-ready patches, secure in the knowledge that they can be quickly pulled back out if it turns out to be a bad idea. Dave Airlie said that sometimes reverting a patch is not simple; other changes may depend on it. Torvalds answered that the process can never be entirely automatic. 

Leemhuis said that the most important task is to establish some real deadlines for regression response; Torvalds suggested writing a documentation patch and getting comments. Leemhuis also asked about whether there is too much immediate backporting of fixes that land in the mainline during the merge window, risking backporting regressions as well. The problem there, Torvalds answered, is that linux-next is not getting enough testing; many of those regressions should never get to the mainline in the first place. 

When asked which subsystems handle regressions well, Leemhuis mentioned the tip tree (which handles x86 and many core-kernel patches) and the block subsystem. Gleixner (the "t" in "tip") said that this responsiveness comes at a price; dealing with regressions takes a lot of time. More problematic, Leemhuis said, are often the "sub-subsystems" that go through more than one level of maintainer. The top-level maintainer may understand the situation, but low-level maintainers may be too far removed from Torvalds to share the same priorities. As a result, fixes can sit in linux-next for too long before landing upstream. 

Torvalds asserted that linux-next is useful for changes aimed at the next merge window, but that it is less useful for fixes. The needed test coverage, he said, just isn't there. So running fixes though linux-next is just a waste of time. Arnd Bergmann protested that he is indeed running daily build tests on linux-next and letting maintainers know when things break. Many of the problems he reports move into the mainline unfixed, though. 

As the session closed, the maintainers in the room affirmed that they find the regression-tracking work useful, and that they would like it to continue.
