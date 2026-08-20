---
title: "Unprivileged filesystem mounts, 2018 edition"
url: https://lwn.net/Articles/755593/
date: "May 30, 2018"
category: "Namespaces-User namespaces"
author: "By Jonathan Corbet May 30, 2018"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
May 30, 2018

The advent of user namespaces and container technology has made it possible to extend more root-like powers to unprivileged users in a (we hope) safe way. One remaining sticking point is the mounting of filesystems, which has long been fraught with security problems. Work has been proceeding to allow such mounts for years, and it has gotten a little closer with the posting of a patch series intended for the 4.18 kernel. But, as an unrelated discussion has made clear, truly safe unprivileged filesystem mounting is still a rather distant prospect — at least, if one wants to do it in the kernel. 

Attempts to make the mount operation safe for ordinary users are nothing new; LWN [covered one patch set](<https://lwn.net/Articles/265220/>) back in 2008\. That work was never merged, but the effort to allow unprivileged mounts [picked up](<https://lwn.net/Articles/652468/>) in 2015, when Eric Biederman (along with others, Seth Forshee in particular) got serious about allowing user namespaces to perform filesystem mounts. The [initial work](<https://lwn.net/Articles/697278/>) was merged in 2016 for the 4.8 kernel, but it was known to not be a complete solution to the problem, so most filesystems can still only be mounted by users who are privileged in the initial namespace. 

Biederman has recently posted [a new patch set](<https://lwn.net/Articles/755498/>) "wrapping up" support for unprivileged mounts. It takes care of a number of details, such as allowing the creation of device nodes on filesystems mounted in user namespaces — an action that is deemed to be safe because the kernel will not recognize device nodes on such filesystems. He clearly thinks that this feature is getting closer to being ready for more general use. 

The plan is not to allow the unprivileged mounting of _any_ filesystem, though. Only filesystem types that have been explicitly marked as being safe for mounting in this mode will be allowed. The intended use case is evidently to allow mounting of filesystems via the FUSE mechanism, meaning that the actual implementation will be running in user space. That should shield the kernel from vulnerabilities in the filesystem code itself, which turns out to be a good thing. 

In a separate discussion, the "syzbot" fuzzing project recently [reported](<https://lwn.net/Articles/755633/>) a problem with the XFS filesystem; syzbot has been doing some fuzzing of on-disk data and a number of bugs have turned up as a result. In this case, though, XFS developer Dave Chinner [explained](<https://lwn.net/Articles/755394/>) that the problem would not be fixed. It is a known problem that only affects an older ("version 4") on-disk format and which can only be defended against at the cost of breaking an unknown (but large) number of otherwise working filesystems. Beyond that, XFS development is focused on the version 5 format, which has checksumming and other mechanisms that catch most metadata corruption problems. 

There was an extensive discussion over whether the XFS developers are taking the right approach, but it took a bit of a diversion after Eric Sandeen [complained](<https://lwn.net/Articles/755634/>) about bugs that involve ""merely mounting a crafted filesystem that in reality would never (until the heat death of the universe) corrupt itself into that state on its own"". Ted Ts'o [pointed out](<https://lwn.net/Articles/755635/>) that such filesystems (and the associated crashes) can indeed come about in real life if an attacker creates one and somehow convinces the system to mount it. He named Fedora and Chrome OS as two systems that facilitate this kind of attack by automatically mounting filesystems found on removable media — USB devices, for example. 

There is a certain class of user that enjoys the convenience of automatically mounted filesystems, of course. There is also the container use case, where there are good reasons for allowing unprivileged users to mount filesystems on their own. So, one might think, it is important to fix all of the bugs associated with on-disk format corruption to make this safe. Chinner [has bad news](<https://lwn.net/Articles/755637/>) for anybody who is waiting for that to happen, though: 

There's little we can do to prevent people from exploiting flaws in the filesystem's on-disk format. No filesystem has robust, exhaustive verification of all it's metadata, nor is that something we can really check at runtime due to the complexity and overhead of runtime checking. 

Many types of corruption can be caught with checksums and such. Other types are more subtle, though; Chinner mentioned linking important metadata blocks into an ordinary file as an example. Defending the system fully against such attacks would be difficult to do, to say the least, and would likely slow the filesystem to a crawl. That said, Chinner [doesn't expect](<https://lwn.net/Articles/755668/>) distributors like Fedora to stop mounting filesystems automatically: ""They'll do that when we provide them with a safe, easy to use solution to the problem. This is our problem to solve, not blame-shift it away."" That, obviously, leaves open the question of how to solve a problem that has just been described as unsolvable. 

To Chinner, the answer is clear, at least in general terms: ""We've learnt this lesson the hard way over and over again: don't parse untrusted input in privileged contexts"". The meaning is that, if the contents of a particular filesystem image are not trusted (they come from an unprivileged user, for example), that filesystem should not be managed in kernel space. In other words, FUSE should be the mechanism of choice for any sort of unprivileged mount operation. 

Ts'o [protested](<https://lwn.net/Articles/755669/>) that FUSE is ""a pretty terrible security boundary"" and that it lacks support for many important filesystem types. But FUSE is what we have for now, and it does move the handling of untrusted filesystems out of the kernel. The fusefs-lkl module (which seems to lack a web site of its own, but is built using the [Linux kernel library](<https://github.com/lkl/linux/>) project) makes any kernel-supported filesystem accessible via FUSE. 

When asked (by Ts'o) about making unprivileged filesystem mounts safe, Biederman [made it clear](<https://lwn.net/Articles/755671/>) that he, too, doesn't expect most kernel filesystems to be safe to use in this mode anytime soon: 

Right now my practical goal is to be able to say: "Go run your filesystem in userspace with fuse if you want stronger security guarantees." I think that will be enough to make removable media reasonably safe from privilege escalation attacks. 

It would thus seem that there is a reasonably well understood path toward finally allowing unprivileged users to mount filesystems without threatening the integrity of the system as a whole. There is clearly some work yet to be done to fit all of the pieces together. Once that is done, we may finally have a solution to a problem that developers have been working on for at least a decade.
