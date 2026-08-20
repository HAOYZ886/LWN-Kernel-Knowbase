---
title: "Development issues part 2: Bug tracking"
url: https://lwn.net/Articles/264231/
date: "January 9, 2008"
category: Debugging
author: "By Jonathan Corbet January 9, 2008"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
January 9, 2008

Once upon a time, free software was a relatively rare commodity, and there was a real novelty in being able to run a free package for a specific purpose. The availability of a free C compiler, for example, was cause for celebration. The fact that said compiler was not always the most reliable program on the system did little to reduce enthusiasm; many of us persisted in irrational endeavors (like trying to use gcc to build the X Window System) despite the occasionally painful (and predictable) consequences. And, in the process, we helped to make both programs more reliable. 

There comes a time, though, when even the most die-hard free software proponent wishes that things would just work. As our software finds its way into more situations where failures are unwelcome (at best), the level of tolerance for bugs is falling. The desire for fewer flaws, however, runs counter to the desire for increasingly capable (and thus more complex) software. Somehow we have to find ways to simultaneously grow our systems and reduce the total number of bugs. To this end, a few projects have been having some interesting discussions on the tracking and fixing of bugs. 

As has been discussed in [this companion article](<https://lwn.net/Articles/263555/>), Eric Raymond has been busily stirring up trouble on the Emacs development list. His point, deemed reasonable by your editor, is that Emacs must adopt a number of relatively modern development practices if it is to have any hope of remaining relevant at all. [One of his key points](<https://lwn.net/Articles/264237/>) is that Emacs needs to have a real bug tracking system. Says Eric: 

Now I consider Emacs: 1100K lines, a COCOMO estimate of over 328 years, and no issue database. I think I think I understand much better now now why the team has only been able to ship one release in five years. Trying to converge on a releasable state with as poor a view of the Emacs bug load as we have must be damn near impossible. 

While some of Eric's suggestions appear to be non-starters - imagine trying to get Richard Stallman to hang out on an IRC channel - the bug tracker suggestion might just go somewhere. Certainly it could only be an improvement for a project of that size to have some sort of idea of what the current list of outstanding bugs looks like. It might even help bring about another Emacs release before the end of the decade. 

Bug trackers are not a magical solution to the bug problem, though; in fact, they can create some problems of their own. The Fedora project, which does have a bug tracker, is currently trying to figure out what to do with the contents of that tracker. It [seems](<https://lwn.net/Articles/264253/>) that said tracker contains over 13,000 bugs, almost 10,000 of which apply to Fedora 7 and later. 

A bug database of this size is simply overwhelming to anybody who tries to do something about it. As a result, Fedora users are filing bugs, only to see nothing happen in response. Not even a "thanks for your report" message. This situation is discouraging for everybody involved, causing Fedora users to give up on reporting bugs and developers to fear looking at the tracker. 

In the Fedora case, there appears to be a near-consensus that the biggest problem is in triaging bug entries. This is not a job which can be automated; somebody has to go through bug submissions, weed out the duplicates, identify those which are really "features," figure out which developer should be notified, etc. Tying bug entries to those found in upstream trackers would be a highly useful bonus. Without this sort of effort, the bug tracker quickly fills with low-quality entries which help nobody. 

For the most part, nobody is doing this job for Fedora now. Red Hat is not paying for a staff member to triage bugs, and the wider community has not filled this gap. In the short term, any sort of solution looks like it will have to come from the community, so the Fedora folks are wondering what can be done to encourage more participation. Simply asking for help is the obvious first step, as is making sure that the process is easy. Then they may consider the tactics adopted by other large projects - Mozilla's policy of expressing its appreciation by sending a T-shirt, for example. 

As an aside, one of the more useful bits of information to come from this discussion was the existence of this family of URLs: 

```
http://bugz.fedoraproject.org/<package-name>
```

Fill in the name, and the result is an immediate list of open bugs for the given package. Thus, for example, a visit to [bugz.fedoraproject.org/gcc](<http://bugz.fedoraproject.org/gcc>) yields a list of compiler bugs. This result can be had directly from bugzilla, of course, but this interface is faster and easier. 

The Fedora developers have discussed a number of related issues, such as whether the Fedora bug database should be separated from the RHEL system and what can be done to make Red Hat better appreciate the value of doing more of its quality assurance work in the Fedora repository. But the core problem is just getting human attention applied to the bug reports. Digging through bug databases is a relatively unglamorous job; it is not an easy path toward rock-star hacker status. But it is an important and relatively easy way to help make free software better. 

Just in time to serve as an example of how well bug management can work, the GNOME project has posted [its annual bugzilla statistics](<https://lwn.net/Articles/264393/>). It seems that over 110,000 GNOME bugs were filed in 2007, almost 109,000 of them were closed. The top bug-closers for the year were: 

> 14254| Andre Klapper   
> ---|---  
> 9800|  Tom Parker   
> 7047 |  Susana Pereira   
> 6882 |  Bruno Boaventura   
> 6649 |  Pedro Villavicencio   
  
It is worth pondering for a moment on the amount of energy required to close over 14,000 bugs in a year - that's almost 40 per day, every day, without a break. This kind of energy _does_ exist within our community, and some projects are putting it to very good use. 

While it is easy to get a contrary impression, the kernel does, in fact, have [a bug tracker](<http://bugzilla.kernel.org/>); there is also, in the form of Natalie Protasevich, somebody who handles the care and feeding of that tracker. But, as a recent episode shows, that still is not always sufficient to actually get the bugs fixed. 

On November 13, 2007, [a bug in the SCSI subsystem](<http://lkml.org/lkml/2007/11/13/250>) was reported to the linux-kernel mailing list. It was put into the tracker as [bug 9370](<http://bugzilla.kernel.org/show_bug.cgi?id=9370>) on the same day. Some developers looked at it over the next few days, but, even though a specific commit which appeared to cause the bug had been identified, no solution was forthcoming. Discussion eventually died out. At least until January 2, when Ingo Molnar decided to stir the pot by posting [a patch](<https://lwn.net/Articles/264267/>) to revert the seemingly guilty commit. At that point the discussion picked up and a reliable way of reproducing the bug was found. The commit which was said to have caused the problem was, in fact, not guilty; it had just caused an older bug to come to light. The discussion did not stop there, though. 

A number of charges went back and forth which do not require discussion here. But one core point is this: as long as the bug report sat in the tracker, nothing much appeared to be happening with it - though, it seems, the SCSI developers had not forgotten it and were trying to figure out what was really going on. But once the problem came back to the linux-kernel list in the form of a brute-force solution, the root cause was found in short order. The key here was bringing the problem to the attention of a wider group of people; the crucial [recipe for reproducing the problem](<https://lwn.net/Articles/264270/>) came from a developer who had not been looking at the problem previously. 

In the kernel context, at least, giving wide exposure to a bug often helps immensely in getting that bug fixed. That is especially true for the sort of hard-to-reproduce bugs which tend to come up in kernel programming. So, while bug trackers are a useful tool for ensuring that problems do not fall through the cracks, it seems that one of the most potent anti-bug tools we have - discussing the problem via a widely-distributed email list - is the same tool we have been using for decades.
