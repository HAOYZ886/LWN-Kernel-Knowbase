---
title: Handling kernel security problems
url: https://lwn.net/Articles/290227/
date: "July 16, 2008"
category: "Development model-Security issues; Security"
author: "By Jonathan Corbet July 16, 2008"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
July 16, 2008

Even the most casual observer of the linux-kernel mailing must have noticed that, in the shadow of the firmware flame war, there is also a heated discussion over the management of security issues. There have also been some [attempts](<https://lwn.net/Articles/290228/>) to turn this local battle into a multi-list, regional conflict. Finding the right way to deal with security problems is difficult for any project, and the kernel is no exception. Whether this discussion will lead to any changes remains to be seen, but it does at least provide a clear view of where the disagreements are. 

Things flared up this time in response to the [2.6.25.10](<http://lwn.net/Articles/288474/>) stable kernel update. The announcement stated that ""any users of the 2.6.25 kernel series are STRONGLY encouraged to upgrade to this release"", but did not say why; none of the patches found in this release were marked as security problems. As it happens, there were security-related fixes in that update; some users are upset that they were not explicitly called out as such. They have reached the point of accusing the kernel developers of hiding security problems. 

These problems, it is said, are fixed with relatively benign-sounding commit messages ("x86_64 ptrace: fix sys32_ptrace task_struct leak," for example) and users are not told that a security fix has been made. This, in turn, is thought to put users at risk because (1) they do not know when they need to apply an update, and (2) there is no clear picture of how many security problems are surfacing in the kernel code. So, as "pageexec" (or "PaX Team") [put it](<https://lwn.net/Articles/290230/>): 

the problem i raised was that there's one declared policy in Documentation/SecurityBugs (full disclosure) yet actual actions are completely different and now Linus even admitted it. the problem arising from such inconsistency is that people relying on the declared disclosure policy will make bad decisions and potentially endanger their users. there're two ways out of this sitution: either follow full disclosure in practice or let the world at large know that you (well, Linus) don't want to. in either case people will adjust their security bug handling processes and everyone will be better off. 

There are two aspects to the charge that the kernel is not following a full disclosure policy: commit messages are said to obscure security fixes, and kernel releases do not highlight the fact that security problems have been fixed. There is an aspect of truth to the first charge, in that Linus will [freely admit](<https://lwn.net/Articles/290231/>) to changing commit logs which discuss security problems too explicitly: 

I literally draw the line at anything that is simply greppable for. If it's not a very public security issue already, I don't want a simple "git log + grep" to help find it. 

That said, I don't _plan_ messages or obfuscate them, so "overflow" might well be part of the message just because it simply describes the fix. So I'm not claiming that the messages can never help somebody pinpoint interesting commits to look at, I'm just also not at all interested in doing so reliably. 

His goal here is clear: make life just a little harder for people who are searching the commit logs for vulnerabilities to exploit. One may argue over whether this policy amounts to hiding security problems, or whether it will be effective in reducing exploits (and plenty of people have shown their willingness to do such arguing), but the fact remains that it _is_ the policy followed by Linus at this time. In his view, the committing of a fix is the disclosure of the problem, and there is no need to be more explicit than that. 

That view extends to the whole security update process found in much of the community. He has no respect for embargo policies or delayed disclosure, and he [criticizes](<https://lwn.net/Articles/290233/>) the "whole security circus" which, in his opinion, emphasizes the wrong thing: 

It makes "heroes" out of security people, as if the people who don't just fix normal bugs aren't as important. 

In fact, all the boring normal bugs are _way_ more important, just because there's a lot more of them. I don't think some spectacular security hole should be glorified or cared about as being any more "special" than a random spectacular crash due to bad locking. 

Beyond that, it is often hard to know which patches are truly security fixes. It has been argued at times that _all_ bugs have security relevance; it's mostly just a matter of figuring out how to exploit them. So explicitly marking security fixes risks taking attention away from all of the other fixes, many of which may also, in fact, fix security issues. Thus, [Linus says](<https://lwn.net/Articles/290235/>): 

If people think that they are safer for only applying (or upgrading to) certain patches that are marked as being security-specific, they are missing all the ones that weren't marked as such. Making them even _believe_ that the magic security marking is meaningful is simply a lie. It's not going to be. 

So why would I add some marking that I most emphatically do not believe in myself, and think is just mostly security theater? 

That said, the stable kernel updates go out with patches which are known to be security fixes. Some people clearly believe that being STRONGLY encouraged to update is not sufficient notification of that fact. It does seem that there has been a trend away from explicit recognition of security issues in the stable releases. The inclusion of CVE numbers was once common; in the 2.6.25 series, only [2.6.25.1](<http://lwn.net/Articles/280734/>), [2.6.25.2](<http://lwn.net/Articles/281224/>), and [2.6.25.5](<http://lwn.net/Articles/285310/>) had such numbers in the changelogs. It is, indeed, true that a straightforward reading of the stable release changelogs will not tell users whether those releases fix relevant security issues. 

There are a number of answers to that complaint too, of course. The real information is in the source code, and that is always public. The fixes in the stable series are unlikely to be all that relevant to most users anyway; they are running distributor kernels which are many months behind even the -stable series and which may (or may not) be affected by a specific problem. In the end, users who are concerned about security issues in their kernels have somebody to turn to: their distributors. Linux distributors [follow disclosure rules](<https://lwn.net/Articles/290241/>) and tend to do a pretty thorough job of fixing the known security problems and propagating those fixes to users. For users who need a high level of long-term support, there are distributors who are more than willing to provide that kind of service for a fee. 

As is often the case, what it really comes down to here is resources. It would be nice if somebody were to follow the patch stream (well over 100 patches/day into the mainline) and identify each one which has security implications. For each patch, this person could then figure out which kernel version was first affected by the vulnerability, obtain a CVE number, and issue a nicely-formatted advisory. But this is a huge job, one which nobody is likely to do in an uncompensated mode for any period of time. So somebody would have to pay for this work. And, to a great extent, that is just what the distributors are doing now - with the nice addition that they backport the fixes into the kernels they support. 

It is worth noting that those distributors have not been doing a whole lot of complaining about how security fixes are handled now. Instead, the complaining has come, primarily, from the maintainers of the out-of-tree [grsecurity](<http://www.grsecurity.net/>) project which, from a suitably cynical point of view, could be seen to benefit from raising the profile of Linux kernel security problems. 

But, regardless of the validity of any such charge, there may be some value in what they are asking. It _is_ good to have a clear sense for what the security problems in a piece of code are. If nothing else, it helps the project itself to understand where it stands with regard to security and whether things are getting better or worse. So it would be nice if the kernel developers could be a bit more diligent and organized in how they track security issues, much like the tracking of regressions has improved over the last couple of years. But this kind of improvement will not happen until somebody decides to put the work into it. Actually putting some time into documenting kernel security issues will accomplish far more than complaining on mailing lists.
