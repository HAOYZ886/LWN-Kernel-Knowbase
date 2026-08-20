---
title: "KS2009: Staging, linux-next, and the development process"
url: https://lwn.net/Articles/357805/
date: "October 21, 2009"
category: "Development model-linux-next; Device drivers-Nouveau; Nouveau; Staging tree"
author: "By Jonathan Corbet October 21, 2009 LWN's 2009 Kernel Summit coverage"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
October 21, 2009

* * *

LWN's [2009 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2009/>)

The final two sessions at the 2009 kernel summit looked at how the development process is working. The first, run by Greg Kroah-Hartman, focused on the roles of the stable, staging, and linux-next trees. 

The stable tree was dealt with in short order: everybody seems to be happy about how this tree (which has been in existence for four years now) is working. Are the stable trees being maintained long enough? The answer appears to be "yes," at least for the people who were in the room. Greg noted that he continues to maintain 2.6.27 because it makes his day job easier; the current plan, evidently, is for Willy Tarreau (the current 2.4 maintainer) to take it over once Greg moves on. There was some talk of improving the information presented on kernel.org so that users have a clear view of what's available and what is still under maintenance. 

Moving on to linux-next, Stephen Rothwell stated that, in the last merge window, 22% of the changes which went into the mainline had not been in linux-next previously. Greg described that number as "scary," but Linus thought that getting 78% coverage in linux-next was actually quite good. That's especially true when considering that much of the remaining patch traffic is quickly-developed fixes for problems noticed once the primary changes went into the mainline. 

Still, Andrew Morton was a bit grumpy about developers who bash out patches and fast-track them right into the mainline, occasionally breaking things. He understands their motivation - developers don't want to sit on the patches for another couple months or so waiting for the next merge window - but he would still like to see people being more careful. He thought it should be possible to detect this kind of behavior automatically and put a stop to it. That's not an idea that was universally loved, though. 

In general, it was asked, is it better to send regression fixes to the mainline right away, or is it better to test them in linux-next for a bit first? Many developers opt for the latter, but then others complain that urgently-needed fixes are not available. Linus suggested that developers should not go overboard with regard to delaying the merging of fixes; sometimes it's better to just send them in right away. 

Moving on to staging: the topic of an "outbound staging" tree was discussed briefly. There appear to be no real objections to the idea; anything that makes it easier to move old code out of the kernel is seen as a good thing. 

There was some talk about the wireless drivers in the staging tree. Their existence is seen as rewarding companies which do not really support the community and taking effort away from attempts to create a real solution to the problem. But the simple fact is that these drivers work now and enable people to use their hardware. John Linville pointed out that Ralink has started funding a developer to work on driver issues, so this situation should improve. 

The strongest discussion, perhaps, was reserved for the topic of the Nouveau driver - a reverse-engineered graphics driver for NVIDIA chipsets. Nouveau is shipped by Fedora, but it is not in the staging tree, which is seen as a violation of the rules. Nouveau is not in staging because the developers involved don't want it to be there. It appears that there are two concerns here; the developers seem to be more forthcoming about one than the other: 

  * Graphics drivers generally involve large amounts of user-space code and a wide interface between user space and the kernel. The Nouveau team would like to retain the ability to change the ABI between the components at will; putting the driver into the staging tree is seen as reducing that flexibility. 

  * One of the things that the Nouveau driver does at startup time is to dump a large binary blob into the hardware. Nobody really knows what that blob (the "voodoo") does. It may be firmware, or it may be something else. In particular, it may be something which is copyrightable on its own and not freely distributable. The blob will be taken out of the driver and treated as firmware, but that, on its own, is not a full solution to the problem. How this issue will be resolved remains unclear. 

There was some criticism of Fedora for shipping a driver that is not in the mainline; there is a good chance that they will not do that again anytime soon. 

The developers talked about the dropping of drivers which don't improve. In particular, it seems that the Android drivers never will get into proper shape and will be dropped from staging in 2.6.33. Google is not going to push them forward, but, interestingly, Qualcomm - not always known for its community participation - is working on getting the hardware drivers for the G1 phone merged. (As an aside in the discussion, Martin Bligh stated that the ChromeOS developers are in a different group, with different procedures, and that they will be much more upstream-oriented than the Android group has been). 

The bottom line is that the staging tree is seen as a successful experiment. It has brought a lot of external code into the mainline, where some of it, at least, has improved quickly. Staging has also turned out to be a good way to get new developers involved in working on the kernel. 

Moving on to the development process in general, the discussion returned to an issue which had been heard a couple of times previously: developers who post a patch will be asked to rework it in ways which greatly expand the scope of the work. For example, the [DRBD](<http://lwn.net/Articles/329543/>) developers have been told that they should really unify the kernel's RAID implementations in order to get their device implementation merged. It was agreed that we need to be more reasonable with these requests, and that they should be seen as advisory most of the time. One thing that could be done is to go ahead and merge the patch before making the request; that lets developers see progress and makes it clear that their patches are not being held hostage. 

Linus said that he usually goes to the kernel summit with at least one issue that he is upset about. But he's relatively happy this year. The merge windows are working well, the process is running smoothly, and there are no specific subsystem trees which he sees as "dark spots." Even behavior on the mailing lists is getting better. 

There was a bit of talk about merge window rules, and the exception which allows the merging of drivers after the -rc1 release. Linus stated that the exception exists because he does not like black-and-white rules and he wants the flexibility to take code when it obviously makes sense to do so. That said, he really wants all new code to show up during the merge window. An occasional driver can come later, but it really should be the exception, and it should not be one of the "horror drivers." 

Linus also likes conditional pull requests, the kind which reads "this code should be safe to go in, but it's not strictly necessary." He may or may not pull the code, depending on what else is going on and his mood, but the flexibility is good and he likes to know that the code is pending even if it does not go in right away. He also requested update email when a tree changes between the sending of a pull request and his actual pulling of it. If a pulled tree doesn't match the pull request, it costs him time to figure out why. 

The merge windows are working well, but the -rc3 and rc4 kernels tend to be too big still. Linus would like to see things quiet down sooner. Please, he asked, don't push unnecessary stuff after -rc1. 

Finally, there was talk about the occasional long patch series which seems to be posted over and over without making any real progress. When there's a huge set of patches, there's always something which can be fixed, so it's hard to get it merged. The answer is to break the series apart into smaller chunks and try to get those in one at a time.
