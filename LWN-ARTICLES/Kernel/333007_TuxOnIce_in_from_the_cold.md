---
title: "TuxOnIce: in from the cold?"
url: https://lwn.net/Articles/333007/
date: "May 13, 2009"
category: Development model; Software suspend
author: "By Jake Edge May 13, 2009"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
May 13, 2009

As flamewars go, the recent linux-kernel thread about [TuxOnIce](<http://tuxonice.org/>) was pretty tame. Likely weary of heated discussions in the past, the participants mostly swore off the flames with a bid to work together on Linux hibernation (i.e. suspend to disk). But, there still seems to be an impediment to that collaboration. The long out-of-tree history for TuxOnIce, combined with lead developer Nigel Cunningham's inability or unwillingness to work with the community means that TuxOnIce could have a bumpy road into the kernel--if it ever gets there at all. 

TuxOnIce, formerly known as suspend2 and swsusp2, is a longstanding out-of-tree solution for hibernation. It has an enthusiastic user community along with some features not available in swsusp, which is the current mainline hibernation code. Some of the advantages claimed by TuxOnIce are support for multiple swap devices or regular files as the suspend image destination, better performance via compressed images and other techniques, saving nearly all of the contents of memory including caches, etc. But its vocal users say that the biggest advantage is that TuxOnIce just works for many--some of whom cannot get the current mainline mechanisms to work. 

Much of the recent mainline hibernation work, generally done by Rafael Wysocki and Pavel Machek, has focused on uswsusp, which moves the bulk of the suspend work to user space. So, the kernel already contains two mechanisms for doing hibernation, leaving no real chance for a third to be added. 

There are clear disagreements about how much and which parts should be in the kernel versus in user space. Machek seems to think that nearly all of the task can be handled in user space, while Cunningham is in favor of the advantages--performance and being able to take advantage of in-kernel interfaces--of an all kernel approach. Wysocki is somewhere in the middle, [outlining](<https://lwn.net/Articles/333144/>) some of the advantages he sees in the in-kernel solution: 

One benefit is that we need not anything in the initrd for hibernation to work. Another one is that we can get superior performance, for obvious reasons (less copying of data, faster I/O). Yet another is simpler configuration and no need to maintain a separate set of user space tools. I probably could find more. 

A bigger disconnect, though, is how to proceed. Cunningham would like to see TuxOnIce merged whole as a parallel alternative to swsusp, with an eye to eventually replacing and removing swsusp. Machek and Wysocki are not terribly interested in replacing swsusp, they would rather see incremental improvements--many coming from the TuxOnIce code--proposed and merged. On the one hand, Cunningham has an entire subsystem that he would like to see merged, while the swsusp folks have a subsystem--used by most distributions for hibernation--to maintain. 

Cunningham recently posted an [RFC for merging TuxOnIce](<http://lwn.net/Articles/332036/>) ""with a view to seeking to get it merged, perhaps in 2.6.31 or .32 (depending upon what needs work before it can be merged) and the willingness of those who matter"". That was met with a [somewhat heated reply](<https://lwn.net/Articles/333063/>) by Machek. But Wysocki was quick to [step in to try to avoid the flames](<https://lwn.net/Articles/333065/>): 

Actually, I see advantages of working together versus fighting flame wars. Please stop that, I'm not going to take part in it this time. 

After Cunningham agreed, the discussion turned to _how_ to work together, which is where it seems to have hit an impasse. Wysocki and Cunningham, at least, see some clear advantages in the TuxOnIce code, but, contrary to Cunningham's wishes, having it merged wholesale is likely not in the cards. Cunningham [describes](<https://lwn.net/Articles/333085/>) his plan as follows: 

I'd like to see use have all three [swsusp, uswsusp, and TuxOnIce] for one or two releases of vanilla, just to give time to work out any issues that haven't been foreseen. Once we're all that there are confident there are no regressions with TuxOnIce, I'd remove swsusp. That's my ideal plan of attack. 

Not surprisingly, Wysocki and Machek see things differently. Machek is [not opposed](<https://lwn.net/Articles/333089/>) to bringing some of TuxOnIce into the mainline: ""If we are talking about improving mainline to allow tuxonice functionality... then yes, that sounds reasonable."" Wysocki lays out an [alternative plan](<https://lwn.net/Articles/333090/>) that is much more in keeping with traditional kernel development strategies: 

So this is an idea to replace our current hibernation implementation with TuxOnIce. 

Which unfortunately I don't agree with. 

I think we can get _one_ implementation out of the three, presumably keeping the user space interface that will keep the current s2disk binaries happy, by merging TuxOnIce code _gradually_. No "all at once" approach, please. 

And by "merging" I mean _exactly_ that. Not adding new code and throwing away the old one. 

But, as Cunningham continues pushing for help in getting TuxOnIce merged alongside swsusp, Wysocki [points out](<https://lwn.net/Articles/333104/>) that it requires a great deal of review to get a huge (10,000+ lines of code) set of patches accepted: ""That would take lot of work and we'd also have to ask many other busy people to do a lot of work for us"". Cunningham seems to be under the misapprehension that kernel hackers will be willing to merge a subsystem that duplicates another without a clear overriding reason. Easing what he sees as a necessary transition from swsusp to TuxOnIce is not likely to be that compelling. 

It is clearly frustrating for Cunningham to have a working solution but be unable to get it into the kernel. But it is a direct result of working out of the tree and then trying to present a solution when the kernel has gone in a different direction. It is a common mistake that folks make when dealing with the kernel community. Ray Lee provides a nice [answer](<https://lwn.net/Articles/333106/>) to Cunningham's frustrations, which points to IBM's device mapper contribution that suffered from a similar reaction. Lee notes that Wysocki has offered extremely valuable assistance: 

He's offering to be the social glue between your code and the forms that are accepted and followed here on LKML. Taking things apart from the whole, finding the pieces that are non-controversial or easily argued for, getting them merged upstream with a user, and then moving on to the rest. 

This way, the external TuxOnIce patch set shrinks and shrinks, until it's eventually gone, with all functionality merged into the kernel in one form or another. 

Is your code better than uswsusp? Almost certainly. This isn't about that. This is about making your code better than what it is today, by going through the existing review-and-merge process. 

At one point, Cunningham [pointed](<https://lwn.net/Articles/333122/>) to the SL*B memory allocators as an example of parallel implementations that are all available in the mainline. Various folks responded that memory allocators are fairly self-contained, unlike TuxOnIce. Furthermore, as Pekka Enberg [notes](<https://lwn.net/Articles/333125/>): ""Yes, so please don't make the same mistake we did. Once you have multiple implementations in the kernel, it's extremely hard to get rid of them."" 

There has been a bit of discussion about the technical aspects of the TuxOnIce patch, mostly centering on the way that it frees up memory to allow enough space to create a suspend image, while still adding the contents of that memory to the suspend image. By relying on existing kernel behavior, which is not necessarily guaranteed for the future, TuxOnIce can save nearly all of the memory contents, whereas swsusp dumps caches and the like to create enough memory to build the suspend image. That means that performance after a resume operation may be impacted as those caches are refilled. Overall, though, the main focus of the discussion has been the way forward; so far, there has been little progress on that front. 

This is not the first time that TuxOnIce has gotten to this point. In its earlier guise as swsusp2, Cunningham made several attempts to get it into the mainline. In March of 2004, Andrew Morton [asked that it be broken down](<https://lwn.net/Articles/75971/>) into smaller, more easily digested, chunks. The [same thing](<https://lwn.net/Articles/113555/>) happened again near the end of 2004 when Cunningham proposed adding swsusp2 in one big code ball. It doesn't end there, either, between then and now the same request has been made; at this point one might guess that Cunningham simply isn't willing to do things that way. 

There is a real danger that the TuxOnIce features that its users like could be lost--or remain out-of-tree--if something doesn't give. Either Cunningham has to recognize that the only plausible way to get TuxOnIce into the kernel is via the normal kernel development path, or someone else has to pick it up and start that process themselves. With no one (other than Cunningham) pushing for its inclusion, there simply is no other way for it to get into the mainline.
