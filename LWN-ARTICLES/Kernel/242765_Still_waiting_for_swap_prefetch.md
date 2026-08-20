---
title: Still waiting for swap prefetch
url: https://lwn.net/Articles/242765/
date: "July 25, 2007"
category: "Development model; Memory management-Swapping"
author: 
---

It has been almost two years since LWN [covered the swap prefetch patch](<http://lwn.net/Articles/153353/>). This work, done by Con Kolivas, is based on the idea that if a system is idle, and it has pushed user data out to swap, perhaps it should spend a little time speculatively fetching that swapped data back into any free memory that might be sitting around. Then, when some application wants that memory in the future, it will already be available and the time-consuming process of fetching it from disk can be avoided. 

The classic use case for this feature is a desktop system which runs memory-intensive daemons (updatedb, say, or a backup process) during the night. Those daemons may shove a lot of useful data to swap, where it will languish until the system's user arrives, coffee in hand, the next morning. Said user's coffee may well grow cold by the time the various open applications have managed to fault in enough memory to function again. Swap prefetch is intended to allow users to enjoy their computers _and_ hot coffee at the same time. 

There is a vocal set of users out there who will attest that swap prefetch has made their systems work better. Even so, the swap prefetch patch has languished in the -mm tree for almost all of those two years with no path to the mainline in sight. Con [has given up](<https://lwn.net/Articles/242768/>) on the patch (and on kernel development in general): 

The window for 2.6.23 has now closed and your position on this is clear. I've been supporting this code in -mm for 21 months since 16-Oct-2005 without any obvious decision for this code forwards or backwards. 

I am no longer part of your operating system's kernel's world; thus I cannot support this code any longer. Unless someone takes over the code base for swap prefetch you have to assume it is now unmaintained and should delete it. 

It is an unfortunate thing when a talented and well-meaning developer runs afoul of the kernel development process and walks away. We cannot afford to lose such people. So it is worth the trouble to try to understand what went wrong. 

Problem #1 is that Con chose to work in some of the trickiest parts of the kernel. Swap prefetch is a memory management patch, and those patches always have a long and difficult path into the kernel. It's not just Con who has run into this: Nick Piggin's [lockless pagecache patches](<http://lwn.net/Articles/147153/>) have been knocking on the door for just as long. The [LWN article on Wu Fengguang's adaptive readahead patches](<http://lwn.net/Articles/155510/>) appeared at about the same time as the swap prefetch article - and that was after your editor had stared at them for weeks trying to work up the courage to write something. Those patches were only merged earlier this month, and, even then, only after many of the features were stripped out. Memory management is not an area for programmers looking for instant gratification. 

There is a reason for this. Device drivers either work or they do not, but the virtual memory subsystem behaves a little differently for every workload which is put to it. Tweaking the heuristics which drive memory management is a difficult process; a change which makes one workload run better can, unpredictably, destroy performance somewhere else. And that "somewhere else" might not surface until some large financial institution somewhere tries to deploy a new kernel release. The core kernel maintainers have seen this sort of thing happen often enough to become quite conservative with memory management changes. Without convincing evidence that the change makes things better (or at least does no harm) in all situations, it will be hard to get a significant change merged. 

In [a recent interview](<http://apcmag.com/6759/interview_with_con_kolivas_part_1_computing_is_boring>) Con stated: 

Then along came swap prefetch. I spent a long time maintaining and improving it. It was merged into the -mm kernel 18 months ago and I've been supporting it since. Andrew [Morton] to this day remains unconvinced it helps and that it 'might' have negative consequences elsewhere. No bug report or performance complaint has been forthcoming in the last 9 months. I even wrote a benchmark that showed how it worked, which managed to quantify it! 

The problem is that, as any developer knows, "no bug reports" is not the same as "no bugs." What is needed in a situation like this is not just testimonials from happy desktop users; there also needs to be some sort of sense that the patch has been tried out in a wide variety of situations. The relatively self-selecting nature of Con's testing community (more on this shortly) makes that wider testing harder to achieve. 

A patch like swap prefetch will require a certain amount of support from the other developers working in memory management before it can be merged. These developers have, as a whole, not quite been ready to jump onto the prefetch bandwagon. A concern which has been raised a few times is that the morning swap-in problem may well be a sign of a larger issue within the virtual memory subsystem, and that prefetch mostly serves as a way of papering over that problem. And it fails to even paper things completely, since it brings back some pages from swap, but doesn't (and really can't) address file-backed pages which will also have been pushed out. The conclusion that this reasoning leads to is that it would be better to find and fix the real problem rather than hiding it behind prefetch. 

The way to address this concern is to try to get a better handle on what workloads are having problems so that the root cause can be addressed. That's why Andrew Morton [says](<https://lwn.net/Articles/242775/>): 

To attack the second question we could start out with bug reports: system A with workload B produces result C. I think result C is wrong for <reasons> and would prefer to see result D. 

and why Nick Piggin [complains](<https://lwn.net/Articles/242776/>): 

Not talking about swap prefetch itself, but everytime I have asked anyone to instrument or produce some workload where swap prefetch helps, they never do. 

Fair enough if swap prefetch helps them, but I also want to look at why that is the case and try to improve page reclaim in some of these situations (for example standard overnight cron jobs shouldn't need swap prefetch on a 1 or 2GB system, I would hope). 

There have been a few attempts to characterize workloads which are improved by swap prefetch, but the descriptions tend toward the vague and hard to reproduce. This is not an easy situation to write a simple benchmark for (though Con has tried), so demonstrating the problem is a hard thing to do. Still, if the prefetch proponents are serious about wanting this code in the mainline, they will need to find ways to better communicate information about the problems solved by prefetch to the development community. 

Communications with the community have been an occasional problem with Con's patches. Almost uniquely among kernel developers, Con chose to do most of his work on his own mailing list. That has resulted in a self-selected community of users which is nearly uniformly supportive of Con's work, but which, in general, is not participating much in the development of that work. It is rare to see patches posted to the ck-list which were not written by Con himself. The result was the formation of a sort of cheerleading squad which would occasionally spill over onto linux-kernel demanding the merging of Con's patches. This sort of one-way communication was not particularly helpful for anybody involved. It failed to convince developers outside of ck-list, and it failed to make the patches better. 

This dynamic became actively harmful when ck-list members (and Con) continued to push for inclusion of patches in the face of real problems. This behavior came to the fore after Con posted the RSDL scheduler. RSDL restarted the whole CPU scheduling discussion and ended up leading to some good work. But some users were reporting real regressions with RSDL and were being told that those regressions were to be expected and would not be fixed. This behavior [soured Linus on RSDL](<http://lwn.net/Articles/226963/>) and set the stage for Ingo Molnar's CFS scheduler. Some (not all) people are convinced that Con's scheduler was the better design, but refusal to engage with negative feedback doomed the whole exercise. Some of Con's ideas made it into the mainline, but his code did not. 

The swap prefetch patches appear to lack any obvious problems; nobody is reporting that prefetch makes things worse. But the ck-list members pushing for its inclusion (often with Con's encouragement) have not been providing the sort of information that the kernel developers want to see. Even so, while a consensus in favor of merging this patch has not formed, there are some important developers who support its inclusion. They include [Ingo Molnar](<https://lwn.net/Articles/242935/>) and [David Miller](<https://lwn.net/Articles/242936/>), who says: 

There is a point at which it might be wise to just step back and let the river run it's course and see what happens. Initially, it's good to play games of "what if", but after several months it's not a productive thing and slows down progress for no good reason. 

If a better mechanism gets implemented, great! We'll can easily replace the swap prefetch stuff at such time. But until then swap prefetch is what we have and it's sat long enough in -mm with no major problems to merge it. 

So swap prefetch may yet make it into the mainline - that discussion is not, yet, done. If we are especially lucky, Con will find a way to get back into kernel development, where his talents and user focus are very much in need. But this sort of situation will certainly come up again. Getting major changes into the core kernel is not an easy thing to do, and, arguably, that is how it should be. If the process must make mistakes, they should probably happen on the side of being conservative, even if the occasional result is the exclusion of patches that end up being helpful.
