---
title: Bisection divides users and developers
url: https://lwn.net/Articles/277872/
date: "April 15, 2008"
category: "Debugging; Development tools-Kernel debugging; Git"
author: "By Jonathan Corbet April 15, 2008"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
April 15, 2008

The last couple of years have seen a renewed push within the kernel community to avoid regressions. When a patch is found to have broken something that used to work, a fix must be merged or the offending patch will be removed from the kernel. It's a straightforward and logical idea, but there's one little problem: when a kernel series includes over 12,000 changesets (as 2.6.25 does), how does one find the patch which caused the problem? Sometimes it will be obvious, but, for other problems, there are literally thousands of patches which could be the source of the regression. Digging through all of those patches in search of a bug can be a needle-in-the-haystack sort of proposition. 

One of the many nice tools offered by the git source code management system is called "bisect." The bisect feature helps the user perform a binary search through a range of patches until the one containing the bug is found. All that is needed is to specify the most recent kernel which is known to work (2.6.24, say), and the oldest kernel which is broken (2.6.25-rc9, perhaps), and the bisect feature will check out a version of the kernel at the midpoint between those two. Finding that midpoint is non-trivial, since, in git, the stream of patches is not a simple line. But that's the sort of task we keep computers around for. Once the midpoint kernel has been generated, the person chasing the bug can build and test it, then tell git whether it exhibits the bug or not. A kernel at the new midpoint will be produced, and the process continues. With bisect, the problematic patch can be found in a maximum of a dozen or so compile-boot-test cycles. 

Bisect is not a perfect tool. If patch submitters are not careful, bisect can create a broken kernel when it splits a patch series. The patch which causes a bug to manifest itself may not be the one which introduced the bug. In the worst case, a developer may merge a long series of patches, finishing with one brief change which enables all the code added previously; in this case, bisect will find the final patch, which will only be marginally useful. If the person reporting the bug is running a distributor's kernel, it may be hard to get that kernel in a form which is amenable to the bisection process. Bisection might require unacceptable downtime on the only (production) system which is affected by the bug. And, of course, the process of checking out, building, booting, and testing a dozen kernels is not something which one fits into a coffee break. It requires a certain determination on the part of the tester and quite a bit of time. 

All of the points above would suggest that requesting a bisection from a user reporting a bug should be done as a last resort. In that context, it is worth looking at the story of a recent bug report which suggests that some observers, at least, think that kernel developers are relying a little too heavily on this tool. An April 9, Mark Lord [reported](<https://lwn.net/Articles/277875/>) a regression in the networking stack; after making a couple of guesses, the network developers [suggested](<https://lwn.net/Articles/277965/>) that the problem be bisected. 

Mark replied that he did not have the time to go through a full bisection, and that he would much rather be provided a list of commits which might be at fault. That list was not forthcoming, though; there were no developers who had an idea of where the problem might be and, as it turns out, the developer who introduced the bug lives in a time zone which caused him to miss the discussion. Mark's [response](<https://lwn.net/Articles/277966/>) was strong: 

Years ago, Linus suggested that he opposed an in-kernel debugger mainly because he preferred that we *think* more about the problems, rather than just finding/fixing symptoms. This 100% reliance upon git-bisect is worse than that. It has people now just tossing regressions into the code left and right, knowing that they can toss all of the testing back at the poor folks whose systems end up not working. 

Andrew Morton also [worries](<https://lwn.net/Articles/277967/>) that developers resort too quickly to a bisection request rather than working with users as was once done. Either that, he says, or developers just ignore the report from the beginning. 

Other developers have answers to these worries, of course. Kernel developers often are not in a position to reproduce a reported bug; it may depend on the specifics of the user's hardware or workload. So they must depend on the user to try things and inform them when a change fixes the problem. Here's [David Miller's view](<https://lwn.net/Articles/277969/>) on how things used to work: 

In fact, this is what Andrew's so-called "back and forth with the bug reporter" used to mainly consist of. Asking the user to try this patch or that patch, which most of the time were reverts of suspect changes. Which, surprise surprise, means we were spending lots of time bisecting things by hand. 

We're able to automate this now and it's not a bad thing. 

The other answer that one hears is that the situation now is much different, with far more users, much more code, and more problems to deal with. The old "back and forth" mode was better suited to smaller user and developer communities; in the current world, things must be done differently. [David Miller](<https://lwn.net/Articles/277970/>) again: 

What people don't get is that this is a situation where the "end node principle" applies. When you have limited resources (here: developers) you don't push the bulk of the burden upon them. Instead you push things out to the resource you have a lot of, the end nodes (here: users), so that the situation actually scales. 

There is another aspect of the problem which is spoken about a bit less frequently: developers must prioritize bug reports and decide which ones to work on. Unlike some projects, the kernel does not have anybody serving in any sort of bug triage role, so, in the absence of a disgruntled and paying customer, most developers make their own decisions on which problems to try to solve. It should not be surprising that problems with the most complete information are the ones which are most likely to be addressed first. 

A bug report with a bisection that fingers a specific commit is a report with very good information, one which is generally easy to resolve. As an example, consider Mark Lord's report again; he did eventually take the time (five hours, apparently) to bisect the problem and [report the results](<https://lwn.net/Articles/277971/>); the bug was found and fixed almost immediately thereafter - despite the fact that the responsible developer was _still_ sleeping on the other side of the planet. 

Even less spoken about is the fact that quite a few problems are one-off occurrences. Somewhere out there in the world, there is a single user who, due to a highly uncommon mixture of hardware and software, experiences a problem which affects (almost) nobody else. Marginal hardware, out-of-tree patches, and overclocking only make the problem worse. Arjan van de Ven's kernel oops summaries are illustrative in this regard; [the statistics for the 2.6.25-rc kernels](<http://kerneloops.org/version.php?start=1638400&end=1671167&version=25-rc>) show that a half-dozen problems account for over half of the reports, while the vast majority of oopses have only a single occurrence. 

Kernel developers have learned that this kind of problem report tends to go away by itself; the affected user finds a way around the issue (or just gives up) and nobody else ever complains. One can well argue that trying to chase down this kind of problem is not a good use of a kernel developer's time. The hard part is figuring out which reports are of this variety. One relatively straightforward way is to wait until reports from other users confirm the problem - or until a sufficiently determined user bisects the problem and provides a commit ID. In this sense, bisection serves as a sort of triage mechanism which requires users to perform enough work to show that the problem is real. 

So the developers do have very good reasons for requesting bisections from users. That said, there is reason to worry that many users will simply stop sending in bug reports. If the only response they can expect is a bisection request (which they may be in no position to answer), they may see no point in reporting bugs at all. Fewer bug reports is not the path toward more solid kernel releases. So, as useful as it is, bisection will have to be a tool of last resort in most cases. The good news is that the development community does seem to understand that; bisection remains just one of the many tools we have for the isolation and solution of problems. 

The not-quite-so-good news is that, as [Al Viro](<https://lwn.net/Articles/277997/>) and [James Morris](<https://lwn.net/Articles/277998/>) have pointed out, the real problem is in the review of code so that fewer bugs are created in the first place. That is not a problem which can be solved with bisection.
