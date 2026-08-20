---
title: "KS2011: Patch review"
url: https://lwn.net/Articles/464298/
date: "October 24, 2011"
category: "Android; Development model-Code review"
author: "By Jonathan Corbet October 24, 2011 2011 Kernel Summit coverage"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
October 24, 2011

* * *

[2011 Kernel Summit](<https://lwn.net/Articles/KernelSummit2011/>) coverage 

The "patch review" session at the 2011 Kernel Summit went off in some interesting directions, but it all started with Rafael Wysocki making the claim that we simply do not do enough patch review. How, he said, can we fix that? And how can we get thorough review of patches? Just getting a Reviewed-by tag, he said, is often not really enough to give confidence in a patch. That led to some discussion of the relative meaning and merit of Reviewed-by and Signed-off-by. The latter is a much stronger statement in the end, and, it was agreed, carries more weight. 

Ted Ts'o talked about the plight of developers who get a lot of detail-oriented review which they duly address. Only later does a subsystem maintainer come by and say that the basic premise of the patch is wrong and that the previous revisions were for nothing. It would be better, he said, if maintainers could perform high-level review of patches [![\[Rafael
Wysocki\]](https://static.lwn.net/images/conf/2011/lc-europe/RafaelWysocki-sm.jpg)](<https://lwn.net/Articles/464315/>) quickly to help developers avoid useless work. Ingo Molnar disagreed, saying that reviewer bandwidth is, in general, the limiting factor in kernel development. So he insists that the code be clean and free of stylistic problems before he will review it at all. 

Another issue raised by Ted is ensuring that problems raised in previous reviews have been addressed in a new revision of a patch set. Tools like Gerrit, he said, can be most helpful in this regard. With Gerrit, it is possible to look at the changes in a patch over time and to track the comments that were made. At his work, use of Gerrit is mandated by policy, and it helps a lot. 

Johannes Berg said that there is more review happening than people think, it's just that a lot of it is invisible. He looks at almost every wireless patch that goes by but, if he sees nothing to complain about, he'll usually just move on quietly. Responding to every patch, even if he has no objections, would increase the amount of email flying around and slow things down. 

Peter Zijlstra complained that he has a whole mailbox full of patches that he needs to review. Alan Cox corrected him: what Peter really has is a mailbox of patches whose developers need to find reviewers for. Trying to do all the review himself is never going to scale; that is a lesson that has been learned many times in the kernel's history. It is necessary to find a way to spread out the work. 

Andrew Morton said that the quality of the reviews and Acked-by responses is not always that great. Sometimes a patch will go out, only to be followed by several Acked-by responses within a few minutes; he knows that all of those people are not reviewing that patch in that time. Some of those responses, he said, are clearly bogus. Peter suggested that "corporate interest" can be behind that kind of coordinated response. But Dave Airlie pointed out that quick acks can also happen if the patch has already been through internal review and the developers involved already know that they are happy with it. James Bottomley said that he'll often ignore reviews that come from the same company as the original patch; he will also ignore a Reviewed-by response that did not contain any actual comments. 

Alan said that part of the problem comes from development that has been contracted out to other companies - often located in China or India - where the contract requires that the resulting code be merged upstream. Ted noted that this is actually an improvement - upstreaming code was once seen as unimportant. But, he said, it would be good to get any in-house review done in public rather than behind closed doors; he acknowledged that this idea can be a hard sell in some companies. Alan observed that many of those patches, in their early versions, are not something that most people would want to see. 

Rafael concluded that if there is a review problem, it is specific to some subsystems. The maintainers of those subsystems will need to figure things out. But, Ted said, a related problem is large patch sets that touch many subsystems. Those patch sets may never get in at all, or they may see review only for some parts of the set. 

Andrew complained about the acceptance of entirely new features into the kernel. Those features often land on his doorstep without much justification, forcing him to ask the developers to explain their motivations. The kernel community, he complained, is not supporting him well. Who can tell him if a given patch makes sense? Mistakes have been made in the past; bad features have been merged and good stuff has been lost. How, he asked, can he find people who know better about the desirability of specific patches? 

Grant Likely suggested the possible creation of an architectural review group that could consider patches as a whole. Andrew said that might help, but it would have to meet frequently enough to respond to patches quickly. Nobody took responsibility for organizing this group, so it would not be surprising if the idea goes no farther. 

#### When "no" becomes "yes"

Alan said that, once upon a time, Linus used to simply say "no" to a lot of patches. If the distributors shipped it anyway for long enough, he would usually give in and merge it anyway. It is not an infallible method, but it provided a good indication of whether a patch is worthwhile or not. Chris Mason observed that the distributors have changed; they no longer want to ship out-of-tree code in their kernels, so this method has lost some of its efficacy. 

Inevitably somebody pointed out an important example of just this process in action, though. Many of the changes in the Android kernel have been blocked from merging into the mainline, but Android (which certainly qualifies as a sort of distribution) shipped it anyway, and it now has many millions of users. What does that say about how we should treat the Android code? 

As one might imagine, the discussion became rather unfocused and fragmented for a while. It came back together when Linus took the microphone and stated that, simply, code that actually is used is the code that is actually worth something. The Android code is certainly being used; the in-kernel code aimed at the same problems is just a vague idea that is worthless in comparison. We should, he said, consider merging suspend blockers as a real option. Even if it truly is crap, we've had crap in the kernel before. The code [![\[Alan Cox\]](https://static.lwn.net/images/conf/2011/lc-europe/AlanCox-sm.jpg)](<https://lwn.net/Articles/464319/>) does not get any better out of tree. Alan Cox agreed that it is probably a good idea to merge that code. The interface is important and has a lot of users; getting the code merged is the best way to fix the implementation. Ingo also agreed, saying that when code has millions of users, we have to say "yes" to it. 

Tim Bird said that there is a project underway in the Consumer Electronics Linux Forum to fix up the Android patches and resubmit them for merging. That news came as a surprise to the others in the room; it seems that there is an independent effort in Linaro to do the same thing. Hopefully one result of this discussion is that these people will find each other and join forces. 

Ted said that, as kernel developers, we often have an inflated idea of our ability to block things by saying "no." We can't stop things that way; they will happen anyway; Jesse Barnes added that we don't have the power to stop product development. Ben Herrenschmidt (and others) noted that, for a lot of hardware vendors, the requirement now is to provide an Android kernel for their products - not a mainline kernel. There is a Linux fork out there, and it is threatening to take over. We are, Ben said, shooting ourselves in the foot by keeping that code out. 

Linus said that it comes down to a marketing issue. Once upon a time, the traditional Linux distributors used to ship a lot of out-of-tree code in their quest to have the best kernel. They have gotten past that phase now and can pat themselves on the back, but they did the same things that Android is doing now. We should, he said, calm down about the whole thing. 

There were no hard conclusions from the session. Given the way things went, though, it would be seriously surprising not to see the Android code seriously considered for merging in the 3.3 or 3.4 time frame. Android has shown staying power, and much of the opposition appears to have moderated over time. That is a surprising conclusion from a session on patch review, but a worthwhile one nonetheless. 

Next: [Development process issues](<https://lwn.net/Articles/464302/>)
