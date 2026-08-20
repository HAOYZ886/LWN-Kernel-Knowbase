---
title: "Re-deprecating sysctl()"
url: https://lwn.net/Articles/247243/
date: "August 29, 2007"
category: "Development model-User-space ABI; Sysctl"
author: "By Jonathan Corbet August 29, 2007"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
August 29, 2007

The `sysctl()` system call allows a suitably-privileged application to tweak various kernel parameters. It is a useful feature which, as it happens, is almost never used. The reason for that is the existence of the `/proc/sys` virtual directory hierarchy which exports the same functionality in a form which is much easier to use. Callers of `sysctl()` have been encouraged to use `/proc/sys` instead for a long time and the addition of new parameters to `sysctl()` is considered to be against the rules. One year ago, `sysctl()` was [removed from the 2.6.19-rc kernels](<http://lwn.net/Articles/204935/>), only to be restored before the final release. 

`sysctl()` is part of the user-space ABI; it is supposed to continue working forever. That is why the attempt to remove it was ultimately rolled back. So it may be surprising to some to see [a new removal attempt](<https://lwn.net/Articles/247237/>) by Eric Biederman. His latest patch adds a new deprecation warning and an entry in the feature removal schedule putting the end of `sysctl()` in September, 2010. Says Eric: 

After adding checking to register_sysctl_table and finding a whole new set of bugs. Missed by countless code reviews and testers I have finally lost patience with the binary sysctl interface. 

The binary sysctl interface has been sort of deprecated for years and finding a user space program that uses the syscall is more difficult then finding a needle in a haystack. Problems continue to crop up, with the in kernel implementation. So since supporting something that no one uses is silly, deprecate sys_sysctl with a sufficient grace period and notice that the handful of user space applications that care can be fixed or replaced. 

Eric's claim is that this interface is so little-used that it is visibly rotting. There is sufficiently little common code between the `sysctl()` and `/proc/sys` implementations that it is easy for the two to diverge. In the long term, he says, the kernel community will do a better job of not breaking applications by getting rid of `sysctl()` in favor of the interface which is actually used and maintained. 

The new patch has, predictably, drawn opposition from developers who do not want to see the user-space ABI broken in this way. Alan Cox has also [suggested](<https://lwn.net/Articles/247233/>) that the deprecation warning approach will not be successful in getting the few remaining users to switch to `/proc/sys`: 

The whole "whine a bit" process simply doesn't work when you are trying to persuade people to move in a non-hobbyist context. They don't want to move, the message is simply an annoyance, their upstream huge package vendor won't change just to deal with it and they'll class it as a regression from previous releases, an incompatibility and file bugs until it goes away. 

Andrew Morton, instead, [is not opposed](<https://lwn.net/Articles/247238/>) to the patch: 

I think it's worth a try. It might take two, three or five years, who knows? If it turns out to be impractical then we we can just change our minds later, no big loss. 

While there is little disagreement with the policy that the user-space ABI should never break, it does seem that there is room for discussion on how that goal might best be met. Unused code has always had a tendency to break accidentally, and `sysctl()` looks to be very close to being entirely unused. One could, presumably, address this problem with some sort of regression test suite - something the kernel could use more of in general. But the maintenance of interfaces which of almost entirely historical interest is not really helpful to Linux users. So, perhaps, there needs to be a way to remove system calls which have fallen into disuse for a long-enough period. Should this patch go through, we shall see whether three years is sufficient warning for such a change or not.
