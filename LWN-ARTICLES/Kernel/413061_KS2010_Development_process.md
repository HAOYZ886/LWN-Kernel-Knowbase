---
title: "KS2010: Development process"
url: https://lwn.net/Articles/413061/
date: "November 3, 2010"
category: Development model
author: "By Jonathan Corbet November 3, 2010 2010 Kernel Summit"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
November 3, 2010

* * *

[2010 Kernel Summit](<https://lwn.net/Articles/KernelSummit2010/>)

As the 2010 Kernel Summit headed toward its end, Linus Torvalds and Andrew Morton were given a session slot to talk about how the development process is working for them. The bottom line: things are going well, but there are a number of things which can be improved. Release numbering is not one of them, though. 

Linus said that, on the whole, he's very happy with how things are working. He was very happy last year, and things have gotten better since. He does not have any major concerns. That said, he does have some pet peeves. 

At the top of his list was pull requests for trees containing commits made within the last hour; those, he said, make him want to kill somebody. There is no excuse for such requests, especially after -rc3 or so - though they may be understandable during the merge window (but he still doesn't like them). He also advised against faking commit dates to get around this issue - something which has apparently "accidentally" happened in the past. A dead giveaway, he says, is when he does a series of pulls and one of the later ones turns out to be a fast-forward merge. 

There were questions about emergency fixes or patches which have been extensively tested in other settings. But Linus's point is that he wants developers to have tested the exact tree that they are asking him to pull. 

Linus would also like to see more Reported-by, Reviewed-by, and Acked-by tags in patches. And, especially, he would like to see those tags from outside people. A bunch of Reviewed-by tags from people in the same company do not have a lot of value; patches really need outside review. On the other hand, he would like fewer embargoed reports of security bugs the day before a major release. That, it turns out, is what delayed the 2.6.36 release by a week. He won't release a kernel with a known root hole. If you plan to ask for an embargo, he would rather just not be told. 

Andrew, instead, said that he has never had any pet peeves - but he did have a few requests. He does a lot less work than he used to; he handles about 1/3 of the patch volume that once went through his tree. But he does still maintain between 100 and 150 subsystem trees. He would like some help, not with all those trees, but with major decisions like the merging of control groups or checkpoint/restart. Merging control groups set the kernel on a multi-year course; he would really like not to have to make those decisions on his own. So he asked maintainers to get out of their subsystem occasionally and look at major patches to other areas of the kernel. 

He seemed a little hurt by some recent requests for a separate tree for memory management patches. Those calls were driven by some bugs found in the `kmap_atomic()` rewrite; the source problem is that code in -mm doesn't appear in linux-next, so it does not get a lot of testing. He understands that his patch management could use improvement; he'll be getting -mm into linux-next shortly. Andrew said that he would like to keep working on the memory management subsystem, an announcement which was received with relief by a number of the people present. 

The problem with memory management, Andrew said, is that it has become incredibly complex; at this point, only Hugh Dickins understands the whole thing. It can be very hard to figure out how things work. What is more worrisome is that this complexity is turning up throughout the kernel. In response, he has been pushing harder and harder to get people to comment their code. Nobody ever pushes back; we all agree that it's needed. But, Andrew said, "we still suck at it." We're going to be working on the kernel for a very long time; code we write now should be seen as a sort of communication to some stranger five years from now. So we need to put more effort into properly documenting our work. 

Linus jumped in with one other pet peeve: he hates having to be the person to say "no." He finds it a lot easier to just grumble and say "OK, just this time," but it makes him unhappy. According to Linus, most subsystem maintainers are too eager to take new features; he would like them to be more selective. There is one easy way to see which trees make him unhappy: look for those which are pulled late in the merge window. He has a tendency to delay making decisions on merges which look ugly, so unpleasant trees pile up until the end. 

Greg Kroah-Hartman jumped in with what was, perhaps, the most pressing development process question: is it time to change the version numbering scheme? Some people are getting tired of the long numbers, and the term "2.6 kernel" does not really mean anything anymore. Alternatives could be going to 3.0 for the next release or using a year-based scheme (so the next kernel would be 2011.0, say). The discussion went on for a while; it became clear that not everybody sees the need for a change. Additionally, there appears to be a desire to see the minor number reach 42. Linus ran a straw poll which was won (42-33) by the "no change" contingent. So the version numbering scheme will not change for now; Greg promised to try again next year. 

Arjan van de Ven asked: what is our biggest priority for the next six months? Linus answered quickly: he wants to see the VFS scalability work merged in the very near future or he will be very unhappy. Al Viro said that he is not happy with all of that work, especially the dentry cache scalability changes. Linus allowed that "the code is interesting," but said there do not appear to be any real alternatives. 

Ted Ts'o said that the memory management and filesystem people need to get together and figure out how the writeback problem will be solved. Andrew seconded that, saying that he has seen some "drive-by patches," but that there does not seem to be a real plan. Christoph Hellwig described that statement as being unfair: the relevant people, he said, have been talking about the problem and working on it. Ted asked if there was a plan, saying that he certainly hadn't seen one. Christoph responded that the developers have an idea of where they want to go. 

There was a brief discussion of the device tree patches. Linus said that he thought device trees were "always a bad idea," that it is far better to simply discover the hardware present on a system. The catch, of course, is that not all hardware supports robust discovery; in that case, there is no alternative to describing the hardware externally. Device trees seem to be the way to go for such situations. 

[Next: Future summits](<https://lwn.net/Articles/413095/>).
