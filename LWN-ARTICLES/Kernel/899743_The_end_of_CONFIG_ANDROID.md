---
title: The end of CONFIG_ANDROID
url: https://lwn.net/Articles/899743/
date: "July 4, 2022"
category: "Android; Build system-Kernel configuration"
author: "By Jonathan Corbet July 4, 2022"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
July 4, 2022

The kernel has thousands of configuration options, many of which can change the kernel's behavior in subtle or surprising ways. Among those options is [`CONFIG_ANDROID`](<https://elixir.bootlin.com/linux/v5.18.8/source/drivers/android/Kconfig>), which one might expect to be relatively straightforward; its description reads, in its entirety: ""Enable support for various drivers needed on the Android platform"". It turns out that this option does more than that, to the surprise of some users. That has led to a plan to remove this option, but that has brought a surprise or two of its own — and some disagreement — as well. 

The discussion started when Alex Xu [reported](<https://lore.kernel.org/all/1656357116.rhe0mufk6a.none@localhost/>) a read-copy-update (RCU) error that was appearing on his system after resuming from suspend. Shortly thereafter, Xu realized that the problem was tied to the fact that his kernel had been built with `CONFIG_ANDROID` enabled; among other things, that option significantly reduces the time that can elapse before RCU starts putting out stall warnings. RCU maintainer Paul McKenney was not entirely sympathetic after this was revealed: 

> And let's face it, the intent and purpose of CONFIG_ANDROID=y is extremely straightforward and unmistakable. So perhaps people not running Android devices but wanting a little bit of the Android functionality should do something other than setting CONFIG_ANDROID=y in their .config files. Me, I am surprised that it took this long for something like this to bite you. 

This response comes from a part of the discussion that does not appear directly in the archives, but can be seen quoted in [Xu's answer](<https://lwn.net/ml/linux-kernel/1656421946.ic03168yc3.none@localhost/>), where he points out that both Debian and Fedora ship kernels with `CONFIG_ANDROID` enabled, since that is the only way to make the binder module available. Xu suggested that the intent of this option is not quite as ""straightforward and unmistakable"" as one might think; the one-line description mentions nothing about changing internal RCU timeout values. ""If major distro vendors are consistently making this 'mistake', then perhaps the problem is elsewhere"". 

Christoph Hellwig was quick to show up with [a patch](<https://lwn.net/ml/linux-kernel/20220629150102.1582425-2-hch@lst.de/>) removing `CONFIG_ANDROID` altogether, describing it as ""obviously a bad idea"". Greg Kroah-Hartman was equally quick to [agree](<https://lwn.net/ml/linux-kernel/YrxvWT%2FaeQnwEv52@kroah.com/>) and queue the patch for the next merge window. Nobody else objected — until Jason Donenfeld [pointed out](<https://lwn.net/ml/linux-kernel/Yrx5Lt7jrk5BiHXx@zx2c4.com/>) that this option has other surprising effects, and that removing it could create problems on Android systems. 

Specifically, both the random-number generator and the [WireGuard](<https://www.wireguard.com/>) VPN tunnel implementation will make changes in response to the system being suspended. The random-number generator will reseed itself after the system resumes, while WireGuard will clear its key material just prior to suspending. Both actions are intended to improve security, but they can be problematic on Android systems due to how power management is handled there. Devices running Android are narcoleptic; they will go to sleep at any opportunity in order to save power. Resetting the random-number generator that frequently is inefficient at best, and clearing the WireGuard keys that often may disrupt communications entirely. To avoid such problems, these actions are not taken if the kernel has been built with `CONFIG_ANDROID`. 

The removal of `CONFIG_ANDROID` also removes that special behavior; this is a change that, Donenfeld feared, could create regressions in the future. He asked Hellwig for either an assurance that these problems would not result, or for an updated version of the patch that fixed those problems. There followed a not-entirely-pleasant discussion over whose responsibility it was to fix any problems, whether that use of `CONFIG_ANDROID` was correct, whether the removal constitutes a user-space ABI regression, and so on. 

Eventually Kroah-Hartman [signaled](<https://lwn.net/ml/linux-kernel/YryFKXsx%2FBgv%2FoBE@kroah.com/>) his agreement with Hellwig. Any problems experienced by Android devices, he said, would be long since found and fixed by the time a patched kernel actually is shipped to such a device, but the change might fix desktop-related problems now. So this change appears to be headed toward the mainline. 

At the core of the debate was the use of `CONFIG_ANDROID` as an indicator that the system will suspend and resume frequently. But, as has been seen, there are many systems with `CONFIG_ANDROID` enabled that do not exhibit that behavior, but which are getting the related changes anyway. There may also be systems that suspend frequently and that should see that behavior, but which are not running Android and do not have `CONFIG_ANDROID` enabled. The consensus seems to be that using `CONFIG_ANDROID` to regulate RCU stall timeouts or cryptographic power-management responses is a bug that is in need of fixing. 

So, for the purposes of random numbers and WireGuard, some other way to indicate that the system will suspend frequently will be needed. There was talk of a new configuration option or a sysfs knob that could be written to from user space, which would allow the behavior to be changed at run time. Your editor's [suggestion](<https://lwn.net/ml/linux-kernel/87a69slh0x.fsf@meer.lwn.net/>) that the kernel could observe actual suspend behavior and do the right thing on its own was fairly quickly dismissed. 

What will happen instead, it seems, is the addition of [a new configuration option](<https://lwn.net/ml/linux-kernel/20220630191230.235306-1-kaleshsingh@google.com/>), called `CONFIG_PM_USERSPACE_AUTOSLEEP`, that prepares the kernel for a system that will suspend frequently and enables the (formerly) Android-specific behavior. This option has a more extensive help text describing what it actually does and warning that it ""should not be enabled just for fun"". The necessary Android changes have already been created, and this appears to be a solution that everybody involved is happy with. 

The way this solution came about could have been better, though. The kernel community works best when developers work together toward a common goal rather than argue over who is doing things incorrectly. That did eventually happen here, but it took some time to get to that point. It took multiple developers to endow `CONFIG_ANDROID` with its somewhat confusing semantics; it is unsurprising that it took more than one person to straighten it out.
