---
title: "Kernel Summit: Development process"
url: https://lwn.net/Articles/94386/
date: "July 21, 2004"
category: Development model
author: 
---

For all practical purposes, the final topic of the 2004 Kernel Summit was the traditional discussion of the development process. This space is usually set aside as the time to pressure Linus for a deadline; often inquiring minds want to know when a feature freeze will happen, but the issue of interest this time was: when will the 2.7 series begin? The answer was somewhat difficult to interpret, but certainly surprising. 

Linus said that he is ready to open 2.7 at about any time; he had originally thought he would do it in June. There are, however, a few concerns which need to be addressed. One is that he wants better synchronization with the ongoing 2.6 tree, so that fixes can be forward ported and interesting developments backported after they stabilize. Then there is the fact that a number of people have grown used to having a BitKeeper tree around for 2.6, and that somehow that should continue after Linus moves on. Linus has also been very happy with how the 2.6 development process has gone, with Andrew Morton vetting patches in his -mm tree and passing on the ones that work for the mainline. 

The end result, says Linus, is that the kernel development process needs "a BK person for Andrew, and an Andrew for me." He then batted his eyes at Alan Cox and asked: "Will you be my Andrew? Will you be mine?" One might have expected Alan to blast a hole in the wall while effecting a hasty exit, but, instead, he agreed - as long as people are not concerned about his continued employment with Red Hat. Andrew, meanwhile, stated that he has no religious problems with BitKeeper, and was planning on picking it up at some point. Thus, he does not necessarily need a "BK person" to help him out. 

At this point, one might naively conclude that everything has been worked out, but things are not destined to be so simple. 

Linus talked about how happy just about everybody is with 2.6. It has been almost two years since the alleged 2.5 feature freeze, but there still is no great pressure to start a new development series. Linus asks: could things just go on the way they are for a while yet, until enough pressure forms to force the 2.7 fork? 

Bdale Garbee pointed out that, in the absence of a 2.7, many people will conclude that 2.6 has not yet stabilized sufficiently. There may be a need to do the fork just to convince people that 2.6 is ready. Alan Cox had a different idea: given that there is not a great deal of stuff to merge into 2.7, perhaps the developers could actually do a six-month release cycle for a change? 

Andrew pointed out that, during the 2.6 process, he and Linus have been merging patches at a rate of about 10MB/month. There is, he says, no reason to believe that things will not continue that way. The traditional stabilization mechanism, where almost no patches are accepted for long periods of time, does not strike him as a good idea. Instead, Andrew would like to see a 2.6 tree which continues to change and evolve, and let the distributors do the final stabilization work. In his vision of the future, the kernel.org kernel will be the most featureful and fastest kernel out there, but it will not necessarily be the most stable. 

The idea here is that restricting changes creates an incredible "patch pressure," which eventually leads to massive amounts of changes going into the kernel suddenly. At that point, things really do become unstable. It is better to keep the flow rate on patches higher; that keeps the developers happy and gets new code out to users quicker. Andrew really believes this: there are, seemingly, very few patches that he is not willing to accept into 2.6 - as long as they make sense and survive testing in -mm. 

These patches include API changes, incidentally. Stable internal kernel APIs have never been guaranteed, but the developers have usually tried to not make big changes during a stable kernel series. That looks to change now. Among other things, it was said that API changes should be merged _before_ an eventual 2.7 fork, since that would make synchronization between the two trees easier. Your editor, who really would like to see _Linux Device Drivers_ not go obsolete before it hits the shelves, finds this idea somewhat dismaying. 

What may happen is that Linus creates a 2.7 tree in the near future, but that tree will be restricted to truly experimental, destabilizing changes. This tree may have no future: if it doesn't work out, or can't be kept in sync with 2.6, it might simply be dropped. Or it could yet develop into 2.8, if that makes sense. 

Nothing was truly resolved in this discussion, which will certainly continue in the hallways and bars over the next few days. But the Linux kernel is maturing, and the development process looks like it will be changing in response. This could be cause for concern, except for one important thing: the 2.6 series really _has_ worked very well. The kernel is in good hands, and that is one thing that looks like it will not change.
