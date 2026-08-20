---
title: Blocking suspend blockers
url: https://lwn.net/Articles/388131/
date: "May 18, 2010"
category: "Android; Power management-Opportunistic suspend"
author: "By Jonathan Corbet May 18, 2010"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
May 18, 2010

When LWN last [looked at suspend blockers](<http://lwn.net/Articles/385103/>) in April, it appeared that this functionality was on a path to be merged into the mainline sometime soon. It may still be on that path, but an extended discussion has muddied the picture somewhat. It is a relatively small and obscure bit of code, but the fate of suspend blockers may have significant implications on how the kernel community deals with external projects in the future. 

Suspend blockers, remember, are tied to the "opportunistic suspend" mode used by the Android system. In this mode, the kernel is placed into a sort of controlled narcolepsy; it will fall asleep (suspend the system) just about anytime that somebody is not actively prodding it. A suspend blocker is a form of prod which can be used to keep the system awake while some sort of important processing is going on. As long as there are suspend blockers outstanding, the system will not suspend. 

There are two aspects to this approach which sit well with the Android developers. One is that they are able to get better power performance (longer battery life) by suspending the entire system whenever nothing is going on. Using normal runtime power management does not give them the same results. The other key point is that opportunistic suspend can happen even when processes are running in user space. In the absence of a suspend blocker, any computation underway is not considered to be important enough to keep the system awake. This behavior is a form of defense against poorly-written applications which might, otherwise, drain a system's battery in a short period of time. 

Suspend blockers have few enthusiastic supporters. Opportunistic suspend seems like a bit of a hack, and the need to put suspend blocker calls into drivers looks invasive. Even if this feature is configured out of most kernels, it looks like it could be a maintenance burden going forward. Even so, most of the developers involved - including almost everybody involved with Linux power management - have concluded that nobody has any better ideas. So, as Matthew Garrett [put it](<https://lwn.net/Articles/388132/>): 

Look, I don't want to sound like I have a one-track mind or anything, but all of these arguments would be significantly more compelling if someone would actually provide a concrete implementation proposal that deals with the set of use-cases that Google's implementation does and which doesn't make anyone cry. Otherwise the immeasurably most likely outcome is that this code gets merged and we get to live with it. 

Requests for alternatives have been posted a number of times in this discussion, but actual proposals have been rather rare. A number of the suspend blocker opponents seem more interested in changing the use case - mandating that all Android applications be well written, for example. The problem is that users will blame the device (rather than that new dog whistle application) if its battery fails to last long enough. So the Android developers must choose between somehow forcing good behavior on all application developers (perhaps losing the "open to all" feature that is at the core of the Android way of doing things) or creating a system robust enough to function with non-ideal applications installed. 

The Android developers have taken the latter approach. In the process, they have made suspend blockers a key part of their platform. Many of the drivers which have been developed for Android have suspend blocker support built into them; they cannot be merged in their current form if the suspend blocker API is not available for them. So the current alternatives are to keep those drivers out, or to hack out the suspend blocker usage before merging them. In the former case, we have more out-of-tree code; in the latter case, we have in-tree drivers which are not actually used, tested, or maintained by anybody. Neither alternative looks good. 

Merging suspend blockers would make it easier to get much of the rest of this code in; as Android developer Brian Swetland [said](<https://lwn.net/Articles/388142/>): 

With wakelock support in the kernel, I'm able to maintain drivers that (provided they meet the normal style, correctness, etc requirements) that both can be submitted to mainline (yay!) and can ship on production hardware as-is (yay!). Porting other linux based environments to hardware like G1, N1, etc becomes that much easier too, which hopefully makes various folks happy. 

This helps get us ever closer to being able to build a production-ready kernel for various android devices "out of the box" from the mainline tree and gets me ever closer to not being in the business of maintaining a bunch of SoC-specific android-something-2.6.# trees, which seriously is not a business I particularly want to be in. 

("Wakelocks" are the old name for suspend blockers). 

Google and Android have taken a lot of grief for their failure to work with upstream and get their code into the mainline kernel. There can be no doubt that their code could have been handled better; had the Android developers worked with the kernel community _before_ shipping this functionality in millions of handsets, perhaps much of this trouble could have been avoided. But that history cannot be rewritten now; not even the secret git "plumbing" commands can make that happen. But we can try to improve the situation going forward. 

The suspend blocker effort looks like a real attempt to do better. The code has not just been posted; it has been through rather more than the usual number of revisions as its developers have put considerable time into trying to address comments which have been made. A failure to merge it would be demoralizing at best. If the development community refuses this attempt to bring Android and the mainline closer, it risks creating an impression of bad faith at best. If we do not accept their code, we really should not complain about them maintaining it outside of the mainline. 

As of this writing, the 2.6.35 merge window is open. What will happen with suspend blockers is anybody's guess. The power management developers are in favor of merging it, but some others have made a fair amount of noise and Linus has not made his feelings known. So it is hard to say whether this long story is about to come to a close or not.
