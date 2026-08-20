---
title: Unreviewed code in 3.11
url: https://lwn.net/Articles/562294/
date: "August 7, 2013"
category: "Development model-Code review; O TMPFILE; Security"
author: "By Jonathan Corbet August 7, 2013"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
August 7, 2013

Kernel development, like development in most free software projects, is built around the concept of peer review. All patches should be reviewed by at least one other developer; that, it is hoped, will catch bugs before they are merged and lead to a higher-quality end result. While a lot of code review does take place in the kernel project, it is also clearly the case that a certain amount of code goes in without ever having been looked at by anybody other than the original developer. A couple of recent episodes bear a closer look; they show why the community values code review and the hazards of skipping it. 

#### O_TMPFILE

The `O_TMPFILE` option to the `open()` system call was pulled into the mainline during the 3.11 merge window; prior to that pull, it had not been posted in any public location. There is no doubt that it provides a useful feature; it allows an application to open a file in a given filesystem with no visible name. In one stroke, it does away with a whole range of temporary file vulnerabilities, most of which are based on guessing which name will be used. `O_TMPFILE` can also be used with the `linkat()` system call to create a file and make it visible in the filesystem, with the right permissions, in a single atomic step. There can be no doubt that application developers will want to make good use of this functionality once it becomes widely available. 

That said, `O_TMPFILE` has been going through a bit of a rough start. It did not take long for Linus to express [concerns](<https://lwn.net/Articles/558949/>) about the new API; in short, there was no way for applications to determine that they were running on a system where `O_TMPFILE` was _not_ supported. A couple of patches later, those issues had been addressed. Since then, a couple of bugs have been found in the implementation; one, [fixed](<http://git.kernel.org/linus/e94bd3490f4ef342801cfc76b33d8baf9ccc9437>) by Zheng Liu, would oops the kernel. Another, [reported](<https://lwn.net/Articles/562296/>) by Andy Lutomirski, corrupts the underlying filesystem through the creation of a bogus inode. Finally, few filesystems actually support this new option at this point, so it is not something that developers can count on having available, even on Linux systems. 

Meanwhile, Christoph Hellwig has [questioned the API](<https://lwn.net/Articles/562297/>) chosen for this feature: 

Why is the useful tmpfile functionality multiplexed over open when it has very different semantics from a normal open? In addition to the flag problems already discussed to death it also just leads to splattering of the code in the implementation [...] 

Christoph suggests that it would have been better to create a new `tmpfile()` system call rather than adding this feature to `open()`. In the end, he has [said](<https://lwn.net/Articles/562298/>), `O_TMPFILE` needs some more time: 

Given all the problems and very limited fs support I'd much prefer disabling O_TMPFILE for this release. That'd give it the needed exposure it was missing by being merged without any previous public review. 

Neither Al Viro (the author of this feature) nor Linus has responded to Christoph's suggestions, leading one to believe that the current plan is to go ahead with the current implementation. Once the `O_TMPFILE` ABI is exposed in the 3.11 release, it will need to be supported indefinitely. It certainly is supportable in its current form, but it may well have come out better with a bit more discussion prior to merging. 

#### Secret security fixes

Russell King's pre-3.11-rc4 pull request does not appear to have been sent to any public list. Based on [the merge commit](<http://git.kernel.org/linus/6d039f8f037fda35da8124f09c4d2bbe55c9a575>) in the mainline, what Russell said about this request was: 

I've thought long and hard about what to say for this pull request, and I really can't work out anything sane to say to summarise much of these commits. The problem is, for most of these are, yet again, lots of small bits scattered around the place without any real overall theme to them. 

Evidently, the fact that eight out of the 22 commits in that request were security fixes does not constitute a "real overall theme." The patches seem like worthwhile hardening for the ARM architecture, [evidently](<https://lwn.net/Articles/562443/>) written in response to disclosures made at the recently concluded [Black Hat USA 2013](<https://www.blackhat.com/us-13/>) event. While most of the patches carry an Acked-by from Nicolas Pitre, none of them saw any kind of public review before heading into the mainline. 

It was not long before Olof Johansson [encountered a number of problems](<https://lwn.net/Articles/562301/>) with the changes, leading to several systems that were unable to boot. LWN reader kalvdans pointed out [a different obvious bug](<https://lwn.net/Articles/562134/>) in the code. Olof suggested that, perhaps, the patches might have benefited from some time in the linux-next repository, but Russell [responded:](<https://lwn.net/Articles/562302/>)

Tell me how I can put this stuff into -next _and_ keep it secret because it's security related. The two things are totally incompatible with each other. Sorry. 

In this case, it is far from clear that much was gained by taking these patches out of the normal review process. The list of distributors rushing to deploy these fixes to users prior to their public disclosure is likely to be quite short, and, in any case, the cure, as was merged for 3.11-rc4, was worse than the disease. As of this writing, neither bug has been fixed in the mainline, though patches exist for both. 

That said, one can certainly imagine scenarios where it might make sense to develop and merge a fix outside of public view. If a security vulnerability is known to be widely exploitable, one wants to get the fix as widely distributed as possible before the attackers are able to develop their exploits. In many cases, though, the vulnerabilities are not readily exploitable, or, as is the case for the bulk of deployed ARM systems, there is no way to quickly distribute an update in any case. In numerous other cases, the vulnerability in question has been known to the attacker community for a long time before it comes to the attention of a kernel developer. 

For all of those cases, chances are high that the practice of developing fixes in secret does more harm than good. As has been seen here, such fixes can introduce bugs of their own; sometimes, those new bugs can be new security problems as well. In other situations, as in the `O_TMPFILE` case, unreviewed code also runs the risk of introducing suboptimal APIs that must then be maintained for many years. The code review practices we have developed over the years exist for a reason; bypassing those practices introduces a whole new set of risks to the kernel development process. The 3.11 development cycle has demonstrated just how real those risks can be.
