---
title: Proposals for Kernel Summit discussions
url: https://lwn.net/Articles/502624/
date: "June 20, 2012"
category: Kernel Summit
author: "By Jake Edge June 20, 2012"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
June 20, 2012

As preparation for this year's Kernel Summit gets underway, a new "more transparent" process is being used to select the 80-100 participants. The Summit will take place August 27-29, just prior to LinuxCon North America in San Diego. Those interested in attending are being asked to describe the technical expertise they will bring to the meeting, as well as to suggest topics for discussion. All of that is taking place on the [ksummit-2012-discuss](<http://lists.linux-foundation.org/mailman/listinfo/ksummit-2012-discuss>) mailing list since the [announcement](<https://lwn.net/Articles/502181/>) on June 14, so it seems worth a look to see what kinds of topics may find their way onto the agenda. 

Development process issues are a fairly common topic at the summit and they figure in a number of the suggestions for this year. One of the hot topics is the role of maintainers with multiple, at least partly related, ideas about discussions in that area. Thomas Gleixner [noted](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000027.html>) a few concerns that he had in a mini-rant: 

So the main questions I want to raise on Kernel Summit are: 

\- How do we cope with the need to review the increasing amount of (insane) patches and their potential integration? 

\- How do we prevent further insanity to known problem spaces (like cpu hotplug) without stopping progress? 

A side question, but definitely related is: 

\- How do we handle "established maintainers" who are mainly interested in their own personal agenda and ignoring justified criticism just because they can? 

As one might guess, that kicked off a bit of a conversation about those problems on the list, but also led several developers to concur about the need to discuss the problems at the summit. Somewhat more diplomatically, Trond Myklebust [suggested](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000066.html>) a related discussion on a possible restructuring of the maintainer's role: 

Currently, the Linux maintainer appears to be responsible for filling all of the traditional roles of software architect, software developer, patch reviewer, patch committer, and software maintainer. 

My question is whether or not there might be some value in splitting out some of these roles, so that we can assign them to different people, and thus help to address the scalability issues that Thomas raised? 

Greg Kroah-Hartman also [wants](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000128.html>) to talk about maintainership and offered to "referee" a discussion. He has some ideas that he [described at LinuxCon Japan](<https://lwn.net/Articles/500443/>) and in a recent [linux-kernel posting](<https://lwn.net/Articles/502486/>) that he thinks "will go a long ways in helping smooth this out". John Linville also [expressed interest](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000154.html>) in that kind of discussion. 

Another area that is generating a lot of interest is the stable tree. Kroah-Hartman is [interested in finding out](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000128.html>) how the process is working for the other kernel developers: 

[...] is it going well for everyone? Are there things we can do differently? How can I kick maintainers who don't mark patches for stable backports in ways that do not harm them too much? How can I convey decisions about the longterm kernel selection process in a better way so that it isn't surprising to people? 

Based on the number of other submissions that mentioned the stable tree, there seems to be a fair amount to discuss. The relationship between the stable tree and the distributions is one fertile area. Kroah-Hartman [said](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000131.html>) that he often has to go "digging through distro kernel trees" to find patches to apply, to which Andrew Morton [suggested](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000139.html>) that the "distro people need a vigorous wedgie" for not making that easier. Various distribution kernel maintainers (e.g. Josh Boyer and Jiri Kosina) agreed that the distributions could do better, but that some discussion of the process would be worthwhile. 

In addition, some discussion of how distributions could better work with the upstream kernel for regression tracking and bug reporting was [proposed](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000011.html>) by Boyer. Kosina [wants to discuss](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000016.html>) the stable review process with an eye toward helping distributions decide which patches to merge into their kernels. Mark Brown is also [interested](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000092.html>) but from the perspective of embedded rather than enterprise distributions. Others also expressed interest in having stable/longterm tree discussions. 

How to track bugs and regressions was a topic [proposed](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000007.html>) by Rafael Wysocki, who has been reporting to the summit on that topic for many years. He was joined by Dave Jones, who would like to [report on bugs and regressions](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000039.html>), both those found by his "trinity" stress-testing tool and ones that have been found in the Fedora kernel over the last year. Like Wysocki, Kosina is also interested in discussing whether the kernel bugzilla is the right tool for tracking bugs and regressions. 

Kernel testing is another area that seems ripe for a discussion. Fengguang Wu would like to [report](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000052.html>) on his efforts to test kernels as each new commit is added: 

And I would like a chance to talk about doing kernel tests in a timely fashion: whenever one pushes new commits to git.kernel.org, build/boot/stress tests will be kicked off and possible errors be notified back to the author within hours. 

This fast develop-test feedback cycle is enabled by running a test backend that is able to build 25000 kernels and runtime test 3000 kernels (assuming 10m boot+testing time for each kernel) each day. Just capable enough to outrace our patch creation rate ;-) 

On an average day, 1-2 build errors are caught in the 160 monitored kernel trees. 

Wu's posting spawned a long thread where various developers described their test setups and what could be done better. Jones [mentioned](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000182.html>) the Coverity scanner in that thread, which led Jason Wessel to [highlight](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000266.html>) Jones's comment as well as give more information on the tool and the kinds of information it can provide. More and better automated kernel testing is definitely on the minds of a lot of potential summit attendees. 

James Bottomley would [like to eliminate "kernel work creation schemes"](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000051.html>), in particular he targeted the amount of code that is needed to support `CONFIG_HOTPLUG`: 

[...] the massive proliferation of __dev... _mem... __cpu... and their ilk are getting out of control. Plus, the amount of memory they save is tiny (a few pages at best) and finally virtually no-one compiles without CONFIG_HOTPLUG, so they're mostly nops anyway. However, for that very case, we've evolved a massive set of tools to beat ourselves up whenever we violate the rules of using these tags. What I'd like to explore is firstly, can we just eliminate CONFIG_HOTPLUG and make it always y (this will clear up the problem nicely) or, failing that, can we just dump the tags and the tools and stop causing work for something no-one cares about. 

There were few defenders of `CONFIG_HOTPLUG=n` in the thread, but he was also interested in finding ways to avoid constructs that lead to a lot of code churn to no good end. In a somewhat similar vein, H. Peter Anvin would like to discuss the [baseline requirements](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000070.html>) for the kernel. Supporting some of the niche uses of Linux (on exotic hardware or with seriously outdated toolchains) creates an ongoing cost for kernel hackers that Anvin would like to see reduced or eliminated. 

Several PCI topics were proposed, including [PCI root bus hotplug issues](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000008.html>) by Yinghai Lu and a [PCI breakout session](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000025.html>) that Benjamin Herrenschmidt suggested. In the latter, Lu's work, some PCI-related KVM issues, cleaning up some PowerPC special cases, and the rework of the PCI hotplug core could all be discussed. As Herrenschmidt put it: "I think there's enough material to keep us busy and a face to face round table with a white board might end up being just the right thing to do". 

Memory management topics also seem popular. Glauber Costa [proposed](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000106.html>) several topics, including kmem tracking and per-memory-control-group kmem memory shrinking, while Hiroyuki Kamezawa [suggested](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000192.html>) memory control group topics. Johannes Weiner is also interested in [talking about a separate memory management tree](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000237.html>) that would supplement the work that Morton does with the -mm tree. The ever-popular memory control group writeback topic was also suggested by Wu and Weiner. 

Srivatsa S. Bhat would like to [present a newcomer's perspective](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000228.html>) on kernel development with an eye toward reducing some of the challenges new developers face. Josef Bacik has a [similar idea](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000239.html>), and would like to discuss how to make it easier for new contributors. In addition to a report on work in the USB subsystem (and USB 3.0 in particular), Sarah Sharp would [like](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000006.html>) to "do a brief readout" about what she learns at AdaCamp in July: 

AdaCamp is a conference focused on gathering tech women together to work on solutions for getting women into open technology fields, and retaining them. I think this would be of interest to the Linux kernel community, since we have very few women kernel developers. I hope to keep this read out focused on positive changes we can make. 

As one can see, these proposals (and many more that were not mentioned) range all over the kernel map. There tends to be a focus on more process and social aspects of the kernel at the summit, mostly because the hardcore technical topics are generally better handled by a more focused group. The summit tries to address global concerns, and there seem to be plenty to choose from.
