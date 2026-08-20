---
title: "A discussion on kernel-maintainer pain points"
url: https://lwn.net/Articles/952146/
date: "November 27, 2023"
category: "Development model-Diversity"
author: "By Jonathan Corbet November 27, 2023 Maintainers Summit"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
November 27, 2023

* * *

[Maintainers Summit](<https://lwn.net/Articles/951847/>)

A regular feature of the Kernel Maintainers Summit is a session where Linus Torvalds discusses the problems that he has been encountering. In recent years, though, there have been relatively few of those problems, so this year he turned things around a bit by [asking the community](<https://lwn.net/ml/ksummit-discuss/CAHk-=whGOUw=YDsPxd9o5M_JqcisE+TjcQQ-=SLYOnHd12D0Fw@mail.gmail.com/>) what problems it was seeing instead. He then addressed them at the Summit in a session covering aspects of the development community, including feedback to maintainers, diversity (or the lack thereof), and more. 

The first question he mentioned was a suggestion that, because he does test builds after acting on pull requests, he is showing a distrust of his maintainers. These builds slowed the process down during the 6.7 merge window, when Torvalds was traveling and doing the builds on a laptop. He answered that he normally does a build after each pull just as a part of his normal workflow. It is not a matter of not trusting maintainers — though he does also like to verify that everything is OK. He also does builds to confirm any conflict resolutions he may have had to do. 

[![\[Linus Torvalds\]](https://static.lwn.net/images/conf/2023/lpc/LinusTorvalds2-sm.png)](<https://lwn.net/Articles/952157/>) Another question had to do with Torvalds's tendency to give mostly negative feedback. Maintainers quickly learn that, if Torvalds responds to a pull-request email, it is usually bad news. He acknowledged that he tends to operate that way, and said that he is not proud of it. In general, he tries to avoid answering email if he doesn't have to. So if a pull happens without problems, he is happy and wants "to say 'I love you'", but he doesn't act on that. As a result, most of his emails are about problems. He did not say that he would try to change that pattern. 

The session was interrupted by a break at this point; on return, Torvalds said that he is quite happy in general. He was in Hawaii for the first week of the merge window, which might ordinarily make things harder. But he got a lot of pull requests early and, despite the 6.7 merge window bringing in the most commits of any merge window in the project's history, it was "pain-free". There was not a single case of a change breaking his machine, which is rare. 

Another question to Torvalds mentioned that the maintainer tree is quite flat, meaning that he pulls directly from a large number of trees rather than from intermediate maintainers who coalesce pulls from multiple subsystems. He agreed that the tree is quite flat. Sometimes that is by his request; there have been cases where having code go through intermediate maintainers has made things more complicated. But the flatness also, he said, suggests that some maintainers don't have the support that they should and are solely responsible for getting work from contributors into the mainline. Some top-level maintainers do far too much work, he said; they should find ways to delegate some of that work to others. 

There was, he said, a complaint that Thorsten Leemhuis, who has taken on the task of tracking regressions, is pushing maintainers to get those regressions fixed. Torvalds found that surprising; he loves having somebody staying on top of problems that way. He can see that it can be annoying to some maintainers, he said, but if Leemhuis weren't doing that job, he would be doing it himself instead. 

Another question had to do with the "bus factor" of having Torvalds in charge of the whole community; what would happen if he were suddenly unable or unwilling to do that work? Torvalds said that things are working so well in the kernel community that his disappearance would be "a momentary distraction"; there would be "some infighting" as the new order was worked out, then work would continue as always. He said that maintainers should talk to him, though, if they think he should be doing his work differently. 

He noted that the kernel community as a whole has hit a plateau in the last five years; the patch volume and number of developers are not growing as they once did. That may suggest, he said, that the community has hit the limits of how far it can scale. Adding more layers of maintainers would help, he said, but it also would not solve all of the issues that are impeding further growth. 

#### Diversity

The last question to be addressed had to do with the gender imbalance in the community and at the Maintainers Summit (which was 100% male) specifically. Torvalds agreed that the situation was not good and "not going in the right direction", then quickly moved on. Dan Williams returned the discussion to this topic a bit later, noting that he had recently had a discussion with the head of the [OpenJS Foundation](<https://openjsf.org/>). When she joined, she was the only female member of the board; now it is much more balanced. She told Williams that the change was effected through direct outreach to potential members; hoping that the problem would get better on its own was not good enough. 

Williams noted that the 2023 election for members of the Linux Foundation Technical Advisory Board was uncontested; only the incumbents ran to keep their seats. That suggests, he said, that the community is missing chances to reach out to people. Torvalds said that Greg Kroah-Hartman used to do that sort of outreach; Kroah-Hartman, in turn, said that Shuah Khan at the Linux foundation is doing a good job of bringing in interns. The problem is that, after learning how to do kernel development, they disappear into companies and are never seen again. 

Williams said that he went to a recent [Black Is Tech conference](<https://blackistechconference.com/>), which featured a slate of all black developers. This would have been a good event for outreach, but nobody was there to recruit for Linux. Torvalds said that the maintainers in the room were not the best people to be doing outreach. Kroah-Hartman again mentioned programs like Outreachy and the Google Summer of Code, which do well at reaching out to potential developers, but which mostly end up providing employees that disappear into companies. Dave Airlie said that he has been able to get a couple of Outreachy interns working on graphics into community-oriented jobs, and they are still contributing. 

Steve Rostedt said that one problem has to do with the demands on women who are successful in the kernel community; they are quickly "asked to join everything" and it burns them out. Ted Ts'o suggested thinking about the different points in the pipeline; people invited to join panels tend to be mid-level developers, but people are dropping out at all levels, suggesting that there is a wider problem. Developers at different levels have different needs, he said. Josef Bacik agreed that the community relies too heavily on the few women that it has; he cited one developer who got burned out and now prefers to just focus on one area and avoids the community. 

Thomas Gleixner pointed to one end of the pipeline by saying that there are few women in computer-science programs; Airlie said that is true of undergraduate programs, but there are more women doing postgraduate work. Christoph Hellwig says that he sees more women at academic events than at community events. 

Bacik said that he does a lot of recruiting for his employer; he goes to events like the [Grace Hopper Celebration](<https://ghc.anitab.org/>) as a way of finding good candidates. The Linux Foundation, he said, could send people to events like this to let developers know that the Linux community exists. 

Konstantin Ryabitsev said that there are good reasons why developers disappear into companies. It is often the only path available; the Linux Foundation is unable to hire them (it employs few developers in general). Not everybody is able to sacrifice their evenings and weekends to do community work, he said. Hellwig suggested looking harder outside of the US and Europe; there are far more women in engineering elsewhere. Discussion on this topic ended with a suggestion from Ts'o to survey Outreachy interns a couple of years after they complete the program and see if they are still working in tech. If not, it would be good to know why; for now, he said, we are only guessing. 

With that, the session (and the Maintainers Summit as a whole) came to an end; the attendees filed off for the obligatory [group photo](<https://lwn.net/Articles/951851/>) before taking some much-needed rest.
