---
title: Security practices
url: https://lwn.net/Articles/572128/
date: "October 29, 2013"
category: Security
author: "By Jonathan Corbet October 29, 2013 2013 Kernel Summit"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
October 29, 2013

* * *

[2013 Kernel Summit](<https://lwn.net/Articles/KernelSummit2013/>)

Kees Cook has been actively trying to improve the security of the Linux kernel for some time. His talk during the plenary day at the 2013 Kernel Summit was split into two parts. The first, on security antipatterns, was the same as the talk he gave at the Linux Security Summit in September; LWN [covered that talk](<https://lwn.net/Articles/569861/#kees>) at the time, so there is no need to repeat that material here. The second half, instead, was a new talk on what a developer should do in response to a security-relevant bug. This talk, he said, was predicated on the assumption that kernel developers had made an ethical choice in favor of fixing flaws; otherwise their response may differ. 

So what are the goals when dealing with a security fix? The wish, of course, is to get the fix out to end users as quickly as possible. If time is available, identifying the severity of the issue can be helpful, but that process is also error-prone. If the bug turns out to be serious, it [![\[Kees Cook\]](https://static.lwn.net/images/conf/2013/lce-ks/KeesCook-sm.jpg)](<https://lwn.net/Articles/572144/>) is worthwhile to try to minimize the time that the public is exposed once the bug has been disclosed. 

If a developer is unsure about the impact of a given bug, the best thing to do is to simply ask. Help is available in two places: the security@kernel.org list (which consists of a small number of kernel developers) and linux-distros@vs.openwall.org, which is made up of representatives from distributors. Mail to the latter list must include the string "`[vs]`" in the subject line to get past the spam filters. Both lists are private. Members of those lists will attempt to handle serious bugs in a coordinated manner. For less serious issues, the best approach is usually to just take the problems directly to the relevant public list. 

When possible, security-related fixes should be tagged for the stable tree; a "`Fixes:`" tag to identify the commit that introduced the problem is also helpful. If possible, the CVE number assigned to the bug should go into the commit changelog; numbers can be assigned by a number of vendors, or from the [oss-security](<http://oss-security.openwall.org/wiki/mailing-lists/oss-security>) mailing list. 

It's worth noting that patience for embargoes is limited in the kernel community. Any problem sent to security@kernel.org can be kept under embargo for a maximum of five days; the limit on linux-distros is ten days. The whole point of the process is to get fixes out to users quickly; developers are sick of long delays in that regard. 

For distributors and manufacturers who are concerned about getting security fixes, Kees had a simple piece of advice: don't bother with tracking CVE numbers. Instead, just pull the entire stable tree and ship everything in it. A lot of security problems will never have CVE numbers assigned to them; if you only take patches with CVEs, you'll miss a lot of important fixes. 

At the end, Dave Jones jumped in to say that he would very much like to know about security bugs that the Trinity tool did not catch; that will help to refine the tests to catch similar problems in the future. Dan Carpenter expressed a similar wish with regard to the smatch utility. It will probably never be possible to find all security bugs automatically, but any progress in that direction seems like a good thing. 

[Next: [Lightning talks](<https://lwn.net/Articles/572129/>)].
