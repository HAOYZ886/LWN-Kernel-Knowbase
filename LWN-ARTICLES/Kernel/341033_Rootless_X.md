---
title: Rootless X
url: https://lwn.net/Articles/341033/
date: "July 15, 2009"
category: Security
author: "By Jonathan Corbet July 15, 2009"
---

By **Jonathan Corbet**  
July 15, 2009

Complexity has long been known to be the enemy of security. A code base which is small and straightforward can be verified, with a reasonable degree of trust, to do what it is supposed to do and only that. As the code becomes more complex and harder to understand, that sort of verification becomes increasingly difficult. For this reason, developers try to separate code requiring privilege from that which doesn't. Any code which can be run in a nonprivileged mode is code which is relatively unlikely to create security problems, and the code which is left can, hopefully, be reviewed to a level sufficient to give the required degree of confidence in its security. 

Now consider the X window system. It is a massive body of code with a relatively small development community. Some of the code is truly ancient and hasn't seen much real developer attention in years. As with most projects, some of the code is of higher quality than the rest. The X server performs a complex task - based on a complicated protocol - for almost every Linux user. Sometimes it is even exposed to the full Internet. And the X server runs as root, with full privileges. The actual number of security problems found in X over the years has been relatively small, but it is not hard to believe that it is more a matter of luck and a lack of attackers than the inherent quality of the X code base. 

Worries about the X server are not new; that is why there has been discussion of running it in a nonprivileged mode for several years. Much of the work which has been done on display graphics has had this goal in mind. All that notwithstanding, Linux distributions still install X as a setuid program. It just has not been possible to enable a nonprivileged X server to get its job done without opening up the system as a whole. 

Those days are just about done. The Moblin project is now [claiming](<https://lwn.net/Articles/341034/>) that it will be the first distribution to ship with a nonprivileged X server. Where Moblin goes, others will certainly follow. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Some [details](<http://lwn.net/Articles/341035/>) of how this work was done were posted to the xorg-devel list by Jesse Barnes at the beginning of July. According to Jesse, finishing out this multi-year job was "pretty easy." It seems that the pieces are in place now - at least for some graphics hardware - to the point that a few hours of work got the job done. It seems like an almost anti-climactic end to such a long-term challenge. But, of course, the work which has made this result possible has been ongoing for a long time. 

The biggest piece is the kernel mode setting (KMS) code which was merged for 2.6.29. Prior to KMS, the X.org server was charged with finding the hardware and driving it directly from user space. Needless to say, this sort of access requires root privilege, since it can easily be used to compromise the system. KMS (and the associated graphical memory management code) turns graphical hardware into something closer to a normal device with a normal kernel driver - albeit a rather complex and specialized sort of "normal" device. The hardware manipulations requiring privilege have been isolated into a relatively small piece of kernel code; they are now separated from the rather larger body of code implementing the X protocol. 

That means that X code accessing the hardware can now be run unprivileged as long as it has the ability to open the appropriate device file. The server must also have the ability to open related device files: input devices, the virtual console, and so on. But that is a problem that has been solved for years; the login process can easily change the ownership of those files so that an unprivileged server can access them but the world as a whole cannot. 

What's left is some detail work. Some of the `ioctl()` calls for direct rendering are currently root-only; Jesse thinks that they can be made generally available in a safe way. There may be some small additions to the driver ABI to allow the final root-only operations to be pushed onto the kernel side. But that's about it. 

Of course, user-mode X will currently only work with Intel chipsets, since those are the only ones with full KMS support at this time. Radeon drivers are acquiring that support quickly, though, and may be able to support no-root operation in the relatively near future. That leaves NVIDIA as the usual odd chipset out of the big three; the current [Nouveau feature matrix](<http://nouveau.freedesktop.org/wiki/FeatureMatrix>) suggests that it will be some time yet before the requisite features are available there. 

It may also be a little while until we see no-root support in more general-purpose distributions. Moblin has a relatively narrow focus on Intel hardware, by virtue of the fact that it's still mostly Intel people who are doing the work. Distributors who need to make things work on whatever hardware happens to be present may approach a change of this magnitude with a bit more caution. Still, X without root is clearly in the future, and the near future at that.
