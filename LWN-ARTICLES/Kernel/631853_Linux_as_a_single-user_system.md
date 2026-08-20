---
title: "Linux as a single-user system"
url: https://lwn.net/Articles/631853/
date: "February 4, 2015"
category: Embedded systems
author: "By Jonathan Corbet February 4, 2015"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
February 4, 2015

Back in the dim and distant past (2001), somebody going by the name "imel" posted [a patch](</2001/0426/a/single-user.php3>) eliminating the concept of users in the kernel and causing everything to run as root. Unsurprisingly, this patch was not taken particularly seriously at the time. But a version of that patch has returned nearly fourteen years later; this time around it has succeeded in generating a bit more discussion. 

The [patch in question](<https://lwn.net/Articles/631857/>) comes from Iulia Manda; it creates a kernel configuration option that removes the concept of multi-user operation. When appropriately configured, the kernel runs every process with user and group IDs of zero and all capability bits set. This option also removes support for a long list of system calls dealing with user and group IDs and capabilities. 

As is often the case with the more eyebrow-raising patches, this one is driven by the kernel "tinification" effort that seeks to shoehorn the kernel into small-memory systems. Such systems are likely to be running a single, dedicated application, perhaps as the init process, and they have little use for many of the features provided by the kernel — including multi-user operation. Configuring that support out saves a bit of memory (about 25KB), making it easier to fit a contemporary kernel on the smallest systems-on-chip. 

Given the nature of the patch, it would not be surprising to see a chorus of opposition on the lists. In fact, the set of opponents consists mainly of Casey Schaufler, who [said](<https://lwn.net/Articles/631862/>): 

Authoritative LSM hooks were loudly rejected in or about 1999. One of the primary reasons they were rejected was because you could use them do exactly what this patch does, which is to remove the basic Linux security policy. If attitudes have changed sufficiently that removing the "classic" security behavior is now deemed acceptable, I propose that we reintroduce the option of authoritative LSM hooks instead. 

"Authoritative LSM hooks" are Linux security module functions that are able to increase the privilege level of a running process. They were [extensively debated back in 2001](</2001/1108/kernel.php3#authhooks>) when the security module mechanism was being developed. Authoritative hooks did not make the cut in the end; they were seen as a significant security risk in their own right. At that time, one Casey Schaufler [criticized the decision](</2001/1108/a/cs-hooks.php3>), seeing it as a selling-out of important functionality to ease the merging of security modules into the kernel. 

Many years later, Casey clearly has not forgotten. He would apparently like to see the single-user option, if it is to be included at all, implemented as a security module using authoritative hooks. The tinification developers are [unimpressed](<https://lwn.net/Articles/631864/>) by that idea, though; reimplementing authoritative hooks would involve reviving an old (and resolved) discussion to get to a possibly inferior solution to the problem they are interested in addressing. So this patch is unlikely to evolve in that direction. 

Beyond that, though, Casey [raised a complaint](<https://lwn.net/Articles/631866/>) that has come up before with regard to tinification patches: ""You are opening the door to creating a 'Linux' kernel that does not behave like Linux."" Many of the patches in this area will, by their basic nature, have that effect; they are, after all, concerned with removing functionality that is not needed for one specific use case. If the kernel is to be suitable for deployment on tiny systems, it will need to be flexible enough to "not behave like Linux" on those systems. 

That is one way of seeing the problem, anyway. Josh Triplett [responded](<https://lwn.net/Articles/631868/>) that it has long been possible to configure functionality out of the kernel and that the single-user patches are nothing special or new: 

So what's this about "not behaving like Linux"? Linux is whatever lives in linux.git; it's a lot more capable these days, and that doesn't just mean *adding* features. The alternative to a tinier Linux isn't a larger Linux, it's non-Linux embedded OSes that behave *nothing* like Linux because they're *not Linux*. 

In the end, it comes down to a couple of questions: can the tinification developers package their changes in a way that the larger development community can accept, and is that community willing to tolerate patches that enable fundamental changes in how the kernel works? The discussion is not new, of course; it [came up at the 2014 Kernel Summit](<https://lwn.net/Articles/608945/>) among other places. It does not look like one that will come to any sort of quick conclusion. But, in the end, if Linux is not able to run well on very small systems, it will likely be pushed aside by a system that _does_ work well in that environment.
