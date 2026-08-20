---
title: "The strange story of the ARM Meltdown-fix backport"
url: https://lwn.net/Articles/749217/
date: "March 15, 2018"
category: "Development model-Stable tree; Security-Meltdown and Spectre"
author: "By Jonathan Corbet March 15, 2018"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
March 15, 2018

Alex Shi's posting of [a patch series](<https://lwn.net/Articles/749218/>) backporting a set of Meltdown fixes for the arm64 architecture to the 4.9 kernel might seem like a normal exercise in making important security fixes available on older kernels. But this case raised a couple of interesting questions about why this backport should be accepted into the long-term-support kernels — and a couple of equally interesting answers, one of which was rather better received than the other. 

The Meltdown vulnerability is most prominent in the x86 world, but it is not an Intel-only problem; some (but not all) 64-bit ARM processors suffer from it as well. The answer to Meltdown is the same in the ARM world as it is for x86 processors: [kernel page-table isolation](<https://lwn.net/Articles/741878/>) (KPTI), though the details of its implementation necessarily differ. The arm64 KPTI patches entered the mainline during the 4.16 merge window. ARM-based systems notoriously run older kernels, though, so it is natural to want to protect those kernels from these vulnerabilities as well. 

When Shi posted the 4.9 backport, stable-kernel maintainer Greg Kroah-Hartman [responded](<https://lwn.net/Articles/749220/>) with a pair of questions: why has a separate backport been done when the Android Common kernel tree already contains the Meltdown work, and what sort of testing has been done on this backport? In both cases, the answer illustrated some interesting aspects of how the ARM vendor ecosystem works. 

#### Android Common and LTS kernels

The [Android Common kernels](<https://source.android.com/devices/architecture/kernel/android-common>) are maintained by Google as part of the Android Open-Source Project; they are meant to serve as a base for vendors to use when creating their device-specific kernels. These kernels start with the long-term support (LTS) kernels, but then add a number of Android-specific features, including the energy-aware scheduling work, features that haven't made it into the mainline for a number of reasons, and more. They also contain backports of important features and fixes, including the Meltdown fixes. 

The Meltdown-fix backport was quite a bit of work, and it has gone through extensive testing in the Android kernel. Kroah-Hartman [worried](<https://lwn.net/Articles/748748/>) that the new backport may not have all of the necessary pieces or have been as extensively validated as the Android work; as such, it may not be something that should appear in the LTS kernels. The analogous effort for x86 should not be an example to follow, he said: 

Yes, we did a horrid hack for the x86 backports (with the known issues that it has, and people seem to keep ignoring, which is crazy), and I would suggest NOT doing that same type of hack for ARM, but go grab a tree that we all know to work correctly if you are stuck with these old kernels! 

The problem with this idea is that not every ARM system is running Android, and pulling from the Android kernel will not work for vendors whose kernels are closer to the mainline. As Mark Brown [put it](<https://lwn.net/Articles/749221/>): 

While that's a very large part of ARM ecosystem it's not all of it, there are also chip vendors and system integrators who have made deliberate choices to minimize out of tree code just as we've been encouraging them to. 

Those vendors would like to have a long-term supported version of the Meltdown mitigations that does not require dragging in all of the other changes that accumulate in the Android kernels. As Brown pointed out, there are increasing numbers of vendors that are doing what the community has been asking for years and staying closer to the mainline. Not providing a proper backport of these important fixes could be seen as breaking the promise that the community has made: run the officially supported stable kernels and you will get the fixes for significant problems. 

There is, thus, a reasonable argument to be made that a proper set of backports for the Meltdown fixes should find its way into the LTS kernels. One little problem remains, though: a proper backport should be known to actually work. 

#### Testing deemed optional

Shi's response to Kroah-Hartman's question about testing was, in its entirety: ""Oh, I have no A73/A75 cpu, so I can not reproduce meltdown bug."" Reproducing the bug on the A73 would be a bit of a challenge, since that processor does not suffer from Meltdown, but A75 does, so asking for testing results on that CPU does not seem entirely out of line. When Kroah-Hartman repeated his request for testing, though, Ard Biesheuvel [responded](<https://lwn.net/Articles/749222/>): 

If ARM Ltd. issues recommendations regarding what firmware PSCI methods to call when doing a context switch, or which barrier instruction to issue in certain circumstances, they do so because a certain class of hardware may require it in some cases. It is really not up to me to go find some exploit code on GitHub, run it before and after applying the patch and conclude that the problem is fixed. Instead, what I should do is confirm that the changes result in the recommended actions to be taken at the appropriate times. 

Upon receipt of that message, Kroah-Hartman [dropped the patch series entirely](<https://lwn.net/Articles/749223/>), complaining that: ""I can't believe we are having the argument of 'Test that your patches actually work'"". He later [added](<https://lwn.net/Articles/749224/>) that if the developers working on the backport don't have both the hardware and the exploit code, ""then someone is doing something seriously wrong"". He urged them to complain to ARM Ltd to get that problem fixed. 

At that point, the conversation stopped. Whether the testing problem is on its way toward a solution has not been revealed. It does seem right that the fixes should be merged into the LTS kernels; otherwise the promises that the community has made regarding those kernels will start to look hollow. But the vendors depending on the LTS kernels also have a right to fixes that somebody has actually bothered to test; anybody who has worked in system software for any period of time knows that just checking for adherence to a specification is no guarantee of a working solution.
