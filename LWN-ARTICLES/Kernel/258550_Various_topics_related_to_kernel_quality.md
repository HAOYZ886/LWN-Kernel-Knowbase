---
title: Various topics related to kernel quality
url: https://lwn.net/Articles/258550/
date: "November 14, 2007"
category: "Development model-Kernel quality"
author: "By Jonathan Corbet November 14, 2007"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
November 14, 2007

Discussions of kernel quality are not a new phenomenon on linux-kernel. It is, indeed, a topic which comes up with a certain regularity, more so than with many other free software projects. The size of the kernel, the rate at which its code changes, and the wide range of environments in which the kernel runs all lead to unique challenges; add in the fact that kernel bugs can lead to catastrophic system failures and you have the material for no end of debate. 

The latest round began when Natalie Protasevich, a Google developer who spends some time helping Andrew Morton track bugs, posted [this list of a few dozen open bugs](<https://lwn.net/Articles/258553/>) which seemed worthy of further attention. Andrew [responded](<https://lwn.net/Articles/258554/>) with his view of what was happening with those bug reports; that view was "no response from developers" in most cases: 

So I count around seven reports which people are doing something with and twenty seven which have been just ignored. 

A number of developers came back saying, in essence, that Andrew was employing an overly heavy hand and that his assertions were not always correct. Regardless of whether his claims are correct, Andrew has clearly touched a nerve. 

He [defended](<https://lwn.net/Articles/258555/>) his posting by raising his often-expressed fear that the quality of the kernel is in decline. This is, he says, something which requires attention now: 

If the kernel _is_ slowly deteriorating then this won't become readily apparent until it has been happening for a number of years. By that stage there will be so much work to do to get us back to an acceptable level that it will take a huge effort. And it will take a long time after that for the kernel to get its reputation back. 

But is the kernel deteriorating? That is a very hard question to answer for a number of reasons. There is no objective standard by which the quality of the kernel can be judged. Certain kinds of problems can be found by automated testing, but, in the kernel space, many bugs can only be found by running the kernel with specific workloads on specific combinations of hardware. A rising number of bug reports does not necessarily indicate decreasing quality when both the number of users and the size of the code base are increasing. 

Along the same lines, as Ingo Molnar [pointed out](<https://lwn.net/Articles/258557/>), a decreasing number of bug reports does not necessarily mean that quality is improving. It could, instead, indicate that testers are simply getting frustrated and dropping out of the development process - a worsening kernel could actually cause the reporting of fewer bugs. So Ingo says we need to treat our testers better, but we also need to work harder at actually measuring the quality of the kernel: 

I tried to make the point that the only good approach is to remove our current subjective bias from quality metrics and to at least realize what a cavalier attitude we still have to QA. The moment we are able to _measure_ how bad we are, kernel developers will adopt in a second and will improve those metrics. Lets use more debug tools, both static and dynamic ones. Lets measure tester base and we need to measure _lost_ early adopters and the reasons why they are lost. 

It is generally true that problems which can be measured and quantified tend to be addressed more quickly and effectively. The classic example is PowerTop, which makes power management problems obvious. Once developers could see where the trouble was and, more to the point, could see just how much their fixes improved the situation, vast numbers of problems went away over a short period of time. At the moment, the kernel developers can adopt any of a number of approaches to improving kernel quality, but they  [PULL QUOTE:  In the absence of objective measurements, developers trying to improve kernel quality are really just groping in the dark.  END QUOTE] will not have any way of really knowing if that effort is helping the situation or not. In the absence of objective measurements, developers trying to improve kernel quality are really just groping in the dark. 

As an example, consider the discussion of the "git bisect" feature. If one is trying to find a regression which happened between 2.6.23 and 2.6.24-rc1, one must conceivably look at several thousand patches to find the one which caused the problem - a task which most people tend to find just a little intimidating. Bisection helps the tester perform a binary search over a range of patches, eliminating half of them in each compile-and-boot cycle. Using bisect, a regression can be tracked down in a relatively automatic way with "only" a dozen or so kernel builds and reboots. At the end of the process, the guilty patch will have been identified in an unambiguous way. 

Bisection works so well that developers will often ask a tester to use it to track down a problem they are reporting. Some people see this practice as a way for lazy kernel developers to dump the work of tracking down their bugs on the users who are bitten by those bugs. Building and testing a dozen kernels is, they say, too much to ask of a tester. Mark Lord, for example, [asserts that most bugs are relatively easy to find](<https://lwn.net/Articles/258559/>) when a developer actually looks at the code; the whole bisect process is often unnecessary: 

I'm just asking that developers here do more like our Top Penguin does, and actually look at problems and try to understand them and suggest fixes to try. And not rely solely on the git-bisect crutch. It's a good crutch, provided the reporter is a kernel developer, or has a lot of time on their hands. But we debugged Linux here for a long time without it. 

On the other hand, some developers see bisection as a powerful tool which has made it easier for testers to actively help the process. David Miller [says](<https://lwn.net/Articles/258561/>): 

Like the internet, this time spent is beneficial because it's pushing the work out to the end nodes. In fact git bisect is an awesome example of the end node principle in action for software development and QA. For the end-user wanting their bug fixed and the developer it's a win win situation because the reporter is actually able to do something proactive which will help get the bug they want fixed faster. 

Returning to original bug list: another issue which came up was the use of mailing lists other than linux-kernel. Some of the bugs had not been addressed because they had never been reported to the mailing list dedicated to the affected subsystem. Other bugs, marked by Andrew as having had no response, had, in fact, been discussed (and sometimes fixed) on subsystem-specific lists. In both situations, the problem is a lack of communication between subsystem lists and the larger community. 

In response, some developers have, once again, called for a reduction in the use of subsystem-specific lists. We are, they say, all working on a single kernel, and we are all interested in what happens with that kernel. Discussing kernel subsystems in isolation is likely to result in a lower-quality kernel. Ingo Molnar [expresses it this way](<https://lwn.net/Articles/258562/>): 

We lose much more by forced isolation of discussion than what we win by having less traffic! It's _MUCH_ easier to narrow down information (by filter by threads, by topics, by people, etc.) than it is to gobble information together from various fractured sources. We learned it _again and again_ that isolation of kernel discussions causes bad things. 

Moving discussions back onto linux-kernel seems like a very hard sell, though. Most subsystem-specific lists feature much lower traffic, a friendlier atmosphere, and more focused conversation. Many subscribers of such lists are unlikely to feel that moving back to linux-kernel would improve their lives. So, perhaps, the best that can be hoped for is that more developers would subscribe to both lists and make a point of ensuring that relevant information flows in both directions. 

David Miller [pointed out](<https://lwn.net/Articles/258563/>) another reason why some bug reports don't see a lot of responses: developers have to choose which bugs to try to address. Problems which affect a lot of users, and which can be readily reproduced, have a much higher chance of getting quick developer attention. Bug reports which end up at the bottom of the prioritized list ("chaff"), instead, tend to languish. The system, says David, tends to work reasonably well: 

Luckily if the report being ignored isn't chaff, it will show up again (and again and again) and this triggers a reprioritization because not only is the bug no longer chaff, it also now got a lot of information tagged to it so it's a double worthwhile investment to work on the problem. 

Given that there are unlikely to ever be enough developers to respond to every single kernel bug report, the real problem comes down to prioritization. Andrew Morton has [a clear idea](<https://lwn.net/Articles/258565/>) of which reports should be handled first: regressions from previous releases. 

If we're really active in chasing down the regressions then I think we can be confident that the kernel isn't deteriorating. Probably it will be improving as we also fix some always-been-there bugs. 

Attention to regressions has improved significantly over the last couple of years or so. They tend to be much more actively tracked, and the list of known regressions is consulted before kernel releases are made. The real problem, according to Andrew, is that any regressions which are still there after a release tend to fall off the list. Better attention to those problems would help to ensure that the quality of the kernel improved over time.
