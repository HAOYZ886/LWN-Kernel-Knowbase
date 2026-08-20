---
title: Haunted by ancient history
url: https://lwn.net/Articles/628531/
date: "January 6, 2015"
category: "Bogomips; Development model-User-space ABI; Wireless extensions"
author: "By Jonathan Corbet January 6, 2015"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
January 6, 2015

Kernel development policy famously states that changes are not allowed to break user-space programs; any patch that does break things will be reverted. That policy has been put to the test over the last week, when two such changes were backed out of the mainline repository. These actions demonstrate that the kernel developers are serious about the no-regressions policy, but they also show what's involved in actually living up to such a policy. 

#### The ghost of wireless extensions

Back in the dark days before the turn of the century, support for wireless networking in the kernel was minimal at best. The drivers that did exist mostly tried to make wireless adapters look like Ethernet cards with a few extra parameters. After a while, those parameters were standardized, after a fashion, behind the "wireless extensions" interface. This `ioctl()`-based interface was never well loved, but it did the job for some years until the developers [painted themselves into a corner](<https://lwn.net/Articles/202838/>) in 2006. Conflicting compatibility issues brought development of that API to a close; the good news was that there was already a plan to supersede it with the then under-development nl80211 API. 

Years later, nl80211 is the standard interface to the wireless subsystem. The wireless extensions, which are now just a compatibility interface over nl80211, have been deprecated for years, and the relevant developers would like to be rid of them entirely. So it was perhaps unsurprising to see [a patch](<http://git.kernel.org/linus/24a0aa212ee2dbe44360288684478d76a8e20a0a>) merged for 3.19 that took away the ability to configure the wireless extensions into the kernel. 

Equally unsurprising, though, would be the flurry of complaints that came shortly thereafter. It seems that the [`wicd`](<https://launchpad.net/wicd>) network manager still uses the wireless extensions API. But, perhaps more importantly, the user-space tools (`iwconfig` for example) that were part of the wireless extensions still use it — and they, themselves, are still in use in countless scripts. So this change looks set to break quite a few systems. As a result, Jiri Kosina posted [a patch reverting the change](<http://git.kernel.org/linus/831a39c241e1254b6ddb8dea3144e77b9bbf44b3>) and Linus accepted it immediately. 

There were complaints from developers that users will never move away from the old commands on their own, and that some pushing is required. But it is not the place of the kernel to do that pushing. A better approach, as Ted Ts'o [suggested](<https://lwn.net/Articles/628540/>), would be: 

[W]hy not hack into the "iw" command backwards compatibility so that if argv[0] is "iwlist" or "iwconfig", it provides the limited subset compatibility to the legacy commands. Then all you need to do is to convince the distributions to set up the packaging rules so that "iw" conflicts with wireless-tools, and you will be able to get everyone switched over to iw after at least seven years. 

Such an approach would avoid breaking user scripts. But it would still take a long time before all users of the old API would have moved over, so the kernel is stuck with supporting the wireless extensions API into the 2020's. 

#### Bogomips

Rather older than the wireless extensions is the concept of "bogomips," an estimation of processor speed used in (some versions of) the kernel for short delay loops. The bogomips value printed during boot (and found in `/proc/cpuinfo`) is only loosely correlated with the actual performance of the processor, but people like to compare bogomips values anyway. It seems that some user-space code uses the bogomips value for its own purposes as well. 

If bogomips deserved the "bogo" part of the name back in the beginning, it has only become more deserving over time. Features like voltage and frequency scaling will cause a processor's actual performance to vary over time. The calculated bogomips value can differ significantly depending on how successful the processor is in doing branch prediction while running the calibration loop. Heterogeneous processors make the situation even more complicated. For all of these reasons, the actual use of the bogomips value in the kernel has been declining over time. 

The ARM architecture code, on reasonably current processors, does not use that value at all, preferring to poll a high-resolution timer instead. On some subarchitectures the calculated bogomips value differed considerably from what some users thought was right, leading to complaints. In response, the ARM developers decided to simply remove the bogomips value from `/proc/cpuinfo` entirely. The patch was [accepted for the 3.12 release](<http://git.kernel.org/linus/9fc2105aeaaf56b0cf75296a84702d0f9e64437b>) in 2013. 

Nearly a year and a half later, Pavel Machek [complained](<https://lwn.net/Articles/628543/>) that the change broke pyaudio on his system. Noting that others had complained as well, he posted a patch reverting the change. It was, he said, a user-space regression and, thus, contrary to kernel policy. 

Reverting this change was not a popular idea in the ARM camp; Nicolas Pitre [tried to block it](<https://lwn.net/Articles/628544/>), saying that ""No setups actually relying on this completely phony bogomips value bearing no links to hardware reality could have been qualified as 'working'."" Linus was [unsympathetic](<https://lwn.net/Articles/628545/>), though, saying that regressions were not to be tolerated and that ""The kernel serves user space. That's what we do."" The change was duly reverted; ARM kernels starting with 3.19 will export a bogomips value again; one assumes the change will make it into the stable tree as well. 

That still leaves the little problem that the bogomips value calculated on current ARM CPUs violates user expectations; people wonder when their shiny new CPU shows as having 6.0 bogomips. Even ARM systems are expected to be faster than that. The problem, [according to Nicolas](<https://lwn.net/Articles/628546/>), is that a constant calculated to help with the timer-based delay loops was being stored as the bogomips value; the traditional bogomips value was no longer calculated at all. There is no real reason, he said, to conflate those two values. So he has posted a patch causing bogomips to be calculated by timing the execution of a tight "do-nothing" loop — the way it was done in the beginning. 

The bogomips value has long since outlived its value for the kernel itself. It is calculated solely for user space, and, even there, its value is marginal at best. As Alan Cox [put it](<https://lwn.net/Articles/628547/>), bogomips is mostly printed ""for the user so they can copy it to tweet about how neat their new PC is"". But, since some software depends on its presence, the kernel must continue to provide this silly number despite the fact that it reflects reality poorly at best. Even a useless number has value if it keeps programs from breaking.
