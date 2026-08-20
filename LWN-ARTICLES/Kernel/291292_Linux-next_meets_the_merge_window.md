---
title: "Linux-next meets the merge window"
url: https://lwn.net/Articles/291292/
date: "July 23, 2008"
category: "Development model-linux-next; Releases-2.6.27; linux-next"
author: "By Jonathan Corbet July 23, 2008"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
July 23, 2008

Recent LWN articles on the linux-next tree have noted that, while this tree has been working well in its role of identifying merge conflicts between subsystem trees, it has not yet been through a full kernel development cycle. 2.6.27 will be the first kernel release where linux-next was in existence for the entire preceding cycle; in theory, everything which goes into 2.6.27 should have been aged in linux-next first. As the end of the 2.6.27 merge window nears, a look at how linux-next has affected the process seems warranted. 

One might think that linux-next maintainer Stephen Rothwell would be able to take a break during the merge window; it should mostly be a matter of watching the linux-next tree drain into the mainline. As it happens, the daily linux-next postings ([example)](<https://lwn.net/Articles/291294/>) suggest a fair amount of scrambling to deal with merge conflicts, build failures, and more. There are a number of reasons for this, one of which being that subsystem trees are merged into the mainline in an order which is completely unrelated to their order in linux-next. Patches which remain in linux-next are being applied to a highly unstable base. 

Another interesting phenomenon has been a fair number of patches appearing in linux-next during the merge window. Some of these are actually patches intended for 2.6.28; once maintainers have dumped their 2.6.27 patches into the mainline, they are starting to acquire stuff for the next time around. Stephen has [asked them not to do that](<https://lwn.net/Articles/291295/>), requesting that 2.6.28 material not be directed toward linux-next until after the 2.6.27-rc1 release. The goal is that linux-next should be nearly empty when 2.6.27-rc1 comes out. 

Other patches, though, are intended for 2.6.27 but simply have not done their time in the linux-next tree. That had led to a certain amount of developer grumpiness at times. It is interesting to note, though, that one of the biggest examples of linux-next avoidance - David Miller's merging of the multiqueue networking code which he had finished writing hours before - has generated relatively few complaints. But various other types of conflicts have generated a steady steam of terse notes from Andrew Morton (who is in the unfortunate position of basing his work on top of linux-next) on how new stuff should have been in linux-next weeks ago. 

Another area of, say, colorful conversation has been around the TTY subsystem, currently been subjected to a much-needed thrashing by Alan Cox. Some developers have been unhappy with Alan for merging code which failed to compile, even though those problems had already been identified in linux-next. Alan, instead, has become irritated with other developers who have surprised him with TTY-layer changes of their own, causing Alan's patches not to apply. Alan has some quaint notions about actually testing his patches, so the resolution of this kind of conflict requires the running of a new set of regression tests and such; after this had happened a few times in a row, he started getting a little short-tempered. These issues would appear to have been worked out at this point, but the idea behind linux-next was to keep them from happening in the first place. 

Yet another source of occasional merge issues is the rebasing of trees. Rebasing, in git-speak, is the process of modifying the commit history in a repository to cause a series of patches to look like they were written against a later version of the code than they really were. Rebasing can be a useful technique; it generates a series of patches which applies cleanly to the current state of the tree without generating a bunch of unsightly merge commits. 

Rebasing can be especially useful in the context of linux-next. If testing turns up a patch which breaks the build, simply committing a fix will leave a period in the history where the kernel cannot be built, and that is bad for people running bisections. With the use of git's history editing features, the offending patch can be fixed in place and all evidence of the mistake disappears. In essence, that embarrassing commit mentioning the Eurasian campaign can be fixed up to properly note that we've always been at war with Eastasia. 

But rebasing a repository changes the history (by design), creating, in the process, an entirely new set of commits. Those commits are new code, to the point that any results from testing the older version may no longer apply. The commits also have new names, so any other developer who was using a version of the repository will be shaken off and unable to merge. Issues related to rebasing have come up a couple of times during the merge window, leading Linus to post a [series](<https://lwn.net/Articles/291302/>) [of](<https://lwn.net/Articles/291303/>) [lectures](<https://lwn.net/Articles/291304/>) on the problems that rebasing can cause. It is clearly a tool which must be used with restraint, but occasional use of rebasing can, in the linux-next context, lead to a better final merge. Finding the right balance is something each developer will have to learn. 

In the end, the merge window remains a bit of an unruly time. The process of channeling the work of several hundred developers into the mainline over a two-week period is unlikely to ever be an entirely smooth experience. But, for all its glitches, the 2.6.27 merge window has been (so far!) easier than 2.6.26. The presence of the linux-next tree almost certainly has something to do with that. This tree's role continues to evolve, but its benefits are starting to be felt.
