---
title: "BSD-style securelevel comes to Linux — again"
url: https://lwn.net/Articles/566169/
date: "September 11, 2013"
category: "Capabilities; Security-UEFI secure boot"
author: "By Jonathan Corbet September 11, 2013"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
September 11, 2013

Most of the hand-wringing over the UEFI secure boot mechanism has long passed; those who want to run Linux on systems with secure boot enabled are, for the most part, able to do so. Things are quiet enough that one might be tempted to believe that the problem is entirely solved. As it happens, though, the core patches that implement the lockdown that some developers think is necessary for proper secure boot support still have not made their way into the mainline. The developer behind that work is still trying to get it merged though; in the process, he has brought back an old idea that was last rejected in 1998. 

By Matthew Garrett's reading of the secure boot requirements, a system running in secure boot mode must not allow any user to change the running kernel; not even root is empowered to do so. Just over one year ago, Matthew posted [a set of patches](<https://lwn.net/Articles/514985/>) that implemented the necessary restrictions. In secure boot mode (as defined by the absence of a new capability called, at that time, `CAP_SECURE_FIRMWARE`), the kernel would not allow the loading of unsigned kernel modules, direct access to I/O ports or I/O memory, or, most controversially, use of the `kexec_load()` system call to reboot directly into a new kernel. As one might expect, not everybody liked this type of restriction, which flies in the face of the longstanding Unix tradition of giving root enough rope to shoot itself in the foot. 

So there were discussions around various aspects of these patches, but one of the biggest problems only came to light later. It seems that there is a fundamental flaw in the capability model: it is nearly impossible to add new capability bits without risking problems with applications that do not know about the new bits. In particular: 

  * Some capability-aware applications work by turning off every capability that they do not think they need. If a new bit is added controlling functionality that such an application uses, it will unknowingly disable a necessary capability and cease to work properly. From the point of view of users of this application, this kind of change constitutes an incompatible ABI change. 

  * Other applications work in a blacklist-oriented mode, turning off capabilities that are known not to be needed. In essence, such an application simply sets the capability mask to zero, then sets the bits corresponding to the capabilities it wants. If some sort of security-related functionality is put behind a new bit that is unknown to this kind of application, that application will leave the capability enabled. That, in turn, could make the application insecure. 

In this case, the biggest risk is that whitelist-style applications would inadvertently turn off `CAP_SECURE_FIRMWARE`, essentially putting themselves into secure boot mode even if the system as a whole is not running in that mode. That could cause things to break in mysterious ways. What it comes down to is that, if one is designing a capability-based system, one really must come up with the full list of needed capabilities at the outset. Back in 1998, when capabilities for Linux were being hashed out, nobody had UEFI secure boot in mind. So there is no relevant capability bit available, and adding one now is not really an option. 

More recently, Matthew posted [a new patch set](<https://lwn.net/Articles/566171/>) that eliminates the new capability. Instead, all of the secure boot restrictions were tied to the existing flag controlling whether unsigned kernel modules can be loaded. Matthew's reasoning was that the restriction on module loading exists to prevent the loading of arbitrary code into the running kernel, so it made sense to lock down any other functionality that might make it possible to evade that restriction. Other developers disagreed, though, saying that they needed the ability to restrict module loading while still allowing other functionality — `kexec_load()` in particular — to be used normally. After some discussion, Matthew backed down and withdrew the patches. 

Eventually he came back with what he called [his final attempt](<https://lwn.net/Articles/566114/>) at providing a kernel lockdown facility that wasn't tied to the secure boot mechanism itself. This time around, we have a new sysfs file at `/sys/kernel/security/securelevel` that accepts any of three values. If it is set to zero (the default), everything works as it always has, with no new restrictions. Setting it to one invokes "secure mode," in which all of the restrictions related to secure boot go into effect. Secure mode is also irrevocable; once it has been enabled, it cannot be disabled (short of compromising the kernel, at which point the battle is already lost). There is also an interesting "permanently insecure" mode obtained by setting `securelevel` to `-1`; the system's behavior is the same as with a setting of zero, but it is no longer possible to change the security level. 

In the UEFI secure boot setting, the bootstrap code would take pains to set `securelevel` to one before allowing any processes to run. That helps to avoid race conditions where the system is subverted before the lockdown can be applied. 

Some readers will, by now, have recognized that "securelevel" looks an awful lot like [the BSD functionality](<http://www.freebsd.org/doc/en/books/faq/security.html#idp76718256>) that goes by the same name; it was clearly patterned after BSD's version. Amusingly, this is not the first time that securelevel has been considered for Linux; there was [an extensive discussion on the subject](<https://lkml.org/lkml/1998/4/6/12>) in early 1998, when Alan Cox was pushing strongly for a securelevel feature. At that time, Linus rejected the feature because he had something much better in mind: capabilities. As is usually the case, Linus won out, and Linux got capabilities instead of securelevel. 

More than fifteen years later, it seems that we might just end up with both mechanisms. Thus far, Matthew's latest patch set has not resulted in many screams of agony, so it might just pass review this time — though, at this point, it is almost certainly too late for 3.12. Meanwhile, Vivek Goyal has posted the first version of a [signed kexec patch set](<https://lwn.net/Articles/566170/>) that would limit `kexec_load()` to signed images. That would allow some useful features (kdump, for example) to continue to work properly in the secure boot environment without leaving `kexec_load()` completely open. That, too, will make the secure boot restrictions a bit more palatable and increase their chances of being merged.
