---
title: "i386 and x86_64: back together?"
url: https://lwn.net/Articles/243704/
date: "July 31, 2007"
category: "Architectures-x86; x86 64"
author: "By Jonathan Corbet July 31, 2007"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
July 31, 2007

The `arch` directory in the kernel source tree contains all of the architecture-specific code. There is a lot of code there, despite years of work by the development community to make things generic whenever possible. There are currently 26 different top-level architectures supported by Linux, many of which contain a number of sub-architectures. Two of those top-level architectures are i386 (the original Linux architecture) and x86_64, which is the 64-bit big brother to i386. There is quite a bit of commonality between those two architectures, and some efforts have been made to share code between them whenever possible. Even so, the source trees for the two architectures remain distinct from each other. 

In the view of some developers, at least, the separation of the two architecture trees is a problem. A bug fix which must be applied to one often is applicable to the other, but it's not clear that all fixes are being made in both places. New features, too, must be added twice. It is relatively easy to break one architecture while working on the other. Developers working on architecture-specific projects - virtualization is mentioned often - end up having to do a lot of work to keep up with two strongly related trees. In response to this kind of pressure, the 32-bit and 64-bit PowerPC architectures were merged into a single architecture tree in 2.6.15, and the general consensus seems to be that it was a good move. But no such merger has happened for the x86 variants. 

That may be about to change, though: Thomas Gleixner and Ingo Molnar recently posted [a patch to merge the two architectures](<http://lwn.net/Articles/242439/>) with a request for comments. This patch is huge: it weighs in at over 9MB and touches 1764 files. It is so tied to the current state of the kernel tree that it can only be reasonably applied to one specific commit point in the git repository. This is not the patch which is meant to be applied, though; its purpose is to show what the final result would look like. If and when the time comes to actually merge this patch, it will be done differently: 

As a next step we plan to generate a gradual, fully bisectable, fully working switchover from the current code to the fully populated arch/x86 tree. It will result in about 1000-2000 commits. 

That is a little intimidating as well. Knowing this, the developers of this patch have gone out of their way to make it possible to apply the change with a high level of confidence. So there will be no code changes associated with the merger: it will be possible to build the exact same kernel image from the source tree before and after the change. 

The patch creates a new architecture called `x86` and moves everything from the two existing architectures over. In the small number of cases where each architecture has an identical copy of the same file, only a single file is created in the new tree. More often, though, the two architectures have a file by the same name in the same place, but their contents differ. In such cases, both files are moved into the new tree with a `_32` or `_64` suffix, depending on where it came from. So, for example, both architectures contain `kernel/ioport.c`; the new `x86` architecture has `ioport_32.c` and `ioport_64.c`. Some simple trickery is then employed to ensure that the correct files for the target architecture are built. 

In many (if not most) cases, there is a great deal of common code in the two files, and that common code is left there. The idea at this stage of the game is to get the two architecture trees together without affecting the resulting kernel; that is probably the only way that such a big change would ever be accepted. Once things have been merged, the opportunities for eliminating duplicated code between individual files will become more apparent - the files will usually be right next to each other. One imagines that an army of code janitors would swoop in to do this work, much of which would be relatively straightforward. Once it's done, we would have a shiny new, merged architecture with duplicated code squeezed out, and everybody would be happy. 

Or maybe not. Andi Kleen has [expressed his opposition](<https://lwn.net/Articles/243714/>) to this change: 

I think it's a bad idea because it means we can never get rid of any old junk. IMNSHO arch/x86_64 is significantly cleaner and simpler in many ways than arch/i386 and I would like to preserve that. Also in general arch/x86_64 is much easier to hack than arch/i386 because it's easier to regression test and in general has to care about much less junk. And I don't know of any way to ever fix that for i386 besides splitting the old stuff off completely. 

Andi, by virtue of being the maintainer of the i386 and x86_64 architectures, has a relatively strong voice in this discussion. His core argument - that splitting the architectures allows lots of legacy issues to be confined to the i386 tree - reflects a common practice in kernel code management. Code which only supports relatively new hardware tends to be a lot cleaner than code which handles older devices as well, but removal of support for hardware which is still in use is frowned upon. So, instead, a new subsystem is created for the newer stuff, with the idea that the legacy code can be supported separately until it withers away. A classic example is the way that serial ATA support was implemented within its own subsystem instead of being an addition to the IDE code. Andi, along with a few others, argues that x86-family processor support should be handled in the same way. 

Most of the participants in the early discussion would appear to disagree with Andi, though. Unlike legacy IDE devices, it is argued, the 32-bit architecture is not going to disappear anytime soon. The number of quirks which are truly unique to the i386 architecture is seen as being relatively small. Linus [argues](<https://lwn.net/Articles/243718/>) that it's easier to carry forward legacy code when it's part of a shared tree than when it's shoved off into a corner. Judging from the conversation which followed the initial posting, there is a near-consensus that the unified tree is the right way to go. 

There were a couple of suggestions that the patch could go directly into 2.6.23, but it is probably just as well that things did not happen that way. 2.6.23 has a lot of new stuff already, and this patch is new. Allowing a cycle for the work to age can only be helpful. Besides, we have not yet seen a repository with those 1000 or so separate commits in it. 

More to the point, though: the real discussion on the merger has not yet happened. To rework two architectures into one over the objections of the maintainer would be an extraordinary step verging on a hostile takeover of the code. Maintainers do not have absolute veto power over patches, but overriding a maintainer on a patch this big is not something which is done lightly. So the developers of the unified x86 architecture patch have one big challenge remaining: they have solved the technical issues nicely, and they have convinced much of the development community that this change should be made. But it would be in the best interests of everybody involved if they could find a way to convince the maintainer of the code they are working with as well.
