---
title: "Two that didn't make it"
url: https://lwn.net/Articles/367787/
date: "December 22, 2009"
category: "AlacrityVM; Development model; Filesystems-Ceph"
author: "By Jonathan Corbet December 22, 2009"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
December 22, 2009

The 2.6.33 merge window has run its course, and a great deal of code has been merged into the mainline. The merge window always seems like a bit of a game of musical chairs, though: when the music stops, at least one project tends to be left conspicuously standing. This time around, two projects were left without a chair in the mainline despite having sent in pull requests: the [Ceph distributed filesystem](<http://lwn.net/Articles/258516/>) and the [AlacrityVM](<http://lwn.net/Articles/345296/>) hypervisor code. 

Often, originators of ignored pull requests are left in silence to wonder why those requests were not acted upon. This time around, Linus [explained the missing pulls](<https://lwn.net/Articles/367788/>): there didn't seem to be enough interest in those features. As he put it: 

The best thing to do is to try to have users that are vocal about the feature, and talk about how great it is. Some advocates for it, in other words. Just a few other people saying "hey, I use this, it's great", is actually a big deal to me. For alacrityvm and cephfs, I didn't have that, or they just weren't loud enough for me to hear. 

Sun CEO Scott McNealy once remarked that free software is like a free puppy. There is some truth to that remark in general, and with respect to code pulled into the kernel in particular. The code itself comes for free, with a nice, GPL-compatible license attached to it. But the kernel maintainers know that this new code is likely to make a few messes around the house and chew up their favorite pair of slippers before it is properly trained. It also must be fed and taken for an occasional visit to the veterinarian for years into the future. So it is important to be sure that, at a minimum, this is a puppy that users actually want. That is why Linus is asking for users to express their support for proposed new features. 

Getting that support can be a bit of a catch-22 situation, though. It takes a dedicated user indeed to grab an in-progress patch and build it into their own kernel; most users will not do that. Life can be easier if distributions package proposed code, giving users a chance to test it out without having to build and install a new kernel, but distributors can get into trouble for doing that. [The recent fuss over Nouveau](<http://lwn.net/Articles/366648/>) was a clear example of unhappiness about shipping out-of-tree kernel code. Similarly, a few years ago, SUSE shipped AppArmor without merging it first, drawing [this complaint](<http://lwn.net/Articles/239964/>) from Andrew Morton: 

Sigh. Please don't put us in this position again. Get stuff upstream before shipping it to customers, OK? It ain't rocket science. 

But getting the customers to request the software - within hearing of Linus Torvalds - before it has been either merged or shipped to them can indeed seem like rocket science at times. 

There has been at least one public request for the merging of Ceph in a future development cycle. The bar may be even higher for AlacrityVM, though. There does not appear to be crowd of users asking for a new set of virtualized device drivers which are meant to be used with an out-of-tree virtualization mechanism. Beyond that, past discussions about this code have been long and heated, with some significant disagreements between AlacrityVM developer Gregory Haskins and (in particular) the KVM developers. 

This history led Ingo Molnar to post [a reminder](<https://lwn.net/Articles/367796/>) of flame wars past and a request that Gregory try harder to work with the KVM development community. Needless to say, this posting has started another extensive discussion, with Gregory [stating](<https://lwn.net/Articles/367797/>) that he has tried hard indeed to work with the other developers, and that, in any case, the current AlacrityVM posting, which consists mostly of drivers, is not relevant to KVM. From there, the discussion moved into whether this work is really necessary, the best approaches to improving I/O performance in virtualized guests, and so on. 

It's not clear that there is an obvious solution to this particular disagreement other than having serious users try out the various solutions and report on what works best. That will be hard to do with an out-of-tree virtualization solution, but the existence of this kind of controversy will only make getting the code into the mainline harder. Linus was [quite clear](<https://lwn.net/Articles/367917/>) on that: 

So when I see another virtualization interface, I want the virtualization people to just argue it out amongst themselves. Thanks to the virtue of me personally not caring one whit about virtualization, I can stand back and just watch the fireworks. 

Which is not to say that I enjoy it (I like the occasional flame-fest, but in order to like them I need to _care_ enough to get fired up about them!). So I just don't want the in-fighting to take place in my tree, so I'd rather see the fighting die out _before_ I actually pull. 

This code was developed by SUSE, which, presumably, wishes to provide AlacrityVM to its customers. This may be one of those situations where the distributor has no choice but to ship the code ahead of mainline integration, just to get the user feedback that shows it's worthwhile. That course has risks: the code may never be merged, or it may suffer incompatible changes on its way into the mainline later on. But the alternative may be to see this code languish on the sideline indefinitely.
