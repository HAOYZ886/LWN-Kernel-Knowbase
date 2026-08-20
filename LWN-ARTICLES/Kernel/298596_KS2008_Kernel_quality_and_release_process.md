---
title: "KS2008: Kernel quality and release process"
url: https://lwn.net/Articles/298596/
date: "September 16, 2008"
category: "Development model-Kernel quality"
author: "By Jonathan Corbet September 16, 2008 LWN's 2008 Kernel Summit coverage"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
September 16, 2008

* * *

LWN's [2008 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2008/>)

The first day of the 2008 kernel summit concluded with two sessions dedicated to the quality of our kernels and the process used to produce them. Arjan van de Ven started off talking about the data acquired by the [Kerneloops project](<http://kerneloops.org/>). In a short period of time, Arjan has accumulated information from tens of thousands of kernel crashes and warnings. From that data, he is able to draw some conclusions about how the kernel fails and how well the developers are doing at fixing problems. 

Initially, Kerneloops worked by grabbing oops reports from the kernel mailing lists. Since then, a number of distributors have added facilities to find oops tracebacks in the kernel logs and ship them off to the project (after obtaining confirmation from the user, of course). This tool is now the source of the vast majority (99%) of the oops reports in the system. One of the things Arjan noted is that many of the biggest problems encountered by users are never reported on the kernel mailing lists; the problem reports one sees there are not indicative of what users are actually running into. 

At any given time, the top ten bugs account for a full 60% of the reports; the top 25 make up 70%. So, while there still appear to be many ways to make a kernel crash, most user problems are caused by a very small number of bugs. Fix those problems, and most users will see their troubles go away. At the other end of the scale, almost half of the bugs are represented by a single report. While some of those reports will be the result of obscure timing-related issues, most of them are more likely to be the result of hardware problems. So a lot of the reported problems do not really require any action from the developers. 

A number of reported bugs result from the utrace code. Utrace is an out-of-tree tracing enhancement shipped by Fedora; it seems that, perhaps, this code still isn't quite ready for prime time. There's also quite a few which are attributable to binary-only modules. 

Linus asked how many developers get the occasional oops reports mailed out by the project; maybe ten people raised their hands. Linus would like to see that report mailed to a lot more people, and the regression reports too. If this information got to more developers, perhaps more bugs would get fixed. 

### Regressions

That was a natural point to move into a discussion of regressions led by Rafael Wysocki. Rafael put up a number of plots of regression counts and associated fixes; by fitting a logarithmic function to regression reports and a line to fixes, he was able to extrapolate the point where the two curves intersect and, in theory, all regressions are fixed. It turns out that recent kernels have been released 1-3 weeks before this point is reached. According to his data, Rafael suggests that the optimal time to release 2.6.27 would be in about three weeks. 

One problem raised by Rafael was that fixes for regressions take far too long to get into the mainline. Some subsystem maintainers like to let regression fixes sit in the linux-next tree for a while. It was pointed out, though, that presence in linux-next did not help find the original regression, so there is unlikely to be any value in letting fixes age there; they should, instead, go straight into the mainline. 

Rafael also noted that some regressions attract no debugging effort at all; it seems that nobody is interested in working on them. It can be disheartening for users to hear nothing about a reported regression at all; somebody should at least tell them why the problem is not being worked on. He also noted that regressions which have been bisected (to identify the change which first caused the problem to happen) tend to get fixed much more quickly. The data from the bisection is undoubtedly useful, but the real benefit probably comes from fingering the guilty party, who then feels the need to get a fix in place. 

Another thing Rafael pointed out is that we have a small core of dedicated testers; most of our regressions are reported by a small, recurring group of people. Perhaps we could recruit some of those people to help with the management of bugs. They could track reports, get more information from users, and harass maintainers to get fixes in place. These people have already shown a certain amount of dedication; giving them this kind of role would let them expand the help they are able to give to the kernel community. 

There was also some talk of trying to track the amount of test coverage the kernel is receiving. There could be some sort of mechanism set up, perhaps tied into Fedora's "smolt" system, to report successful boots of the kernel on specific hardware. There are obvious privacy issues which would have to be addressed, and the whole thing would take a certain amount of work. It is not clear that anybody feels this idea is important enough to put the requisite amount of time into. 

### Release process

Matt Mackall asked a question: what would happen if we were to cut the merge window down to one week - merging less code - and shorten the development cycle to match? With some discipline, maybe we could produce a stable kernel release every six weeks. Linus responded that he would love to see this happen. His main motivation was to reduce the size of the -rc1 releases, which have gotten quite big in recent development cycles. A smaller -rc1 would be easier to debug and should, hopefully, stabilize more quickly. 

Quite a bit of time went into discussing this idea. The shorter merge window was clearly worrisome to some developers who feel that the two-week window is already painfully short. Merging of trees with dependencies on other trees would get harder. It would also be harder to get good testing coverage, since there would be less time for testers to play with each release. Some code simply takes a long time to fix; it's not clear that this stabilization could be compressed into the shorter cycle. There would have to be some higher barriers to ensure that code which does get in through a particular merge window is truly ready. 

Andrew Morton jumped in with a complaint about code that shows up in the mainline, but which has never made an appearance in linux-next or the -mm tree. He acknowledged that this would always happen, but asserted that it should be an extraordinary event. The guilty subsystem maintainer, he says, should at least make excuses for doing this. Much of the problem, it was said, comes from vendors who show up with last-minute patches that they want to see merged. The answer was to tell them that it is too late, that the merge window is for subsystem maintainers, not for vendors. 

Getting back to the shorter cycle, Linus pointed out that it would require a great deal of care from everybody involved, especially the first time around. It would require a development cycle which does not start with a lot of pending code - a problem, since there is always a big pile of patches waiting by the time the merge window opens. 

Al Viro suggested only merging a subset of subsystem trees in any development cycle, only accepting trivial patches from the rest. James Bottomley responded that, if his trees lost out in a given development cycle, his definition of "trivial" would surely change. Another suggestion was to simply merge linux-next, but Linus did not like that. He goes out of his way to limit the amount of code he merges each day as a favor to the people to test the nightly repository snapshots. Pulling in all of linux-next would make that impossible. Yet another option is to only pull trees for which the pull request is in place before the merge window opens. This idea seemed popular for a while. 

Just about when it looked like a consensus for trying the idea was settling into place, Matthew Wilcox stated that he didn't like it. His work involves tracking down performance issues, a process which can take quite a bit of time. A shortened development cycle would not allow the time needed to get that work done. Andrew Morton said that he saw no real point in the change; it wasn't addressing any of our biggest problems, and we would lose economies of scale in testing large numbers of changes. Dave Airlie said it would require testers to do twice as much work, dealing with -rc1 kernels twice as often. Ben Herrenschmidt worried that the tighter deadlines would make developers rush, leading to lower-quality code. And Dave Jones said that changing the cycle would make future kernel releases less predictable, making communications with vendors and customers harder. 

These comments essentially ended the discussion of the shorter development cycle idea. In the end, concluded Linus, it was better not to mess with something which isn't completely broken. So nothing may have come with it, but it was an interesting exploration of how things could be done differently.
