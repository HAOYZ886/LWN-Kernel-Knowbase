---
title: Firmware signing
url: https://lwn.net/Articles/645823/
date: "May 27, 2015"
category: "Device drivers-Firmware loading; Firmware"
author: "By Jonathan Corbet May 27, 2015"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
May 27, 2015

The kernel has had [the ability](<https://lwn.net/Articles/525592/>) to enforce signature requirements on loadable modules for a few years now. But there are other types of code loaded by the kernel that are not, yet, subject to such checks; firmware loaded into controllers via the kernel is perhaps the primary example. Work is being done to add the ability to enforce signature requirements for firmware blobs, but not everybody is convinced that there is a need for this feature. 

Luis Rodriguez [described](<https://lwn.net/Articles/645824/>) his plan on various kernel mailing lists. The kernel's module loader is currently being [reworked](<https://lwn.net/Articles/644649/>) (by David Howells) to move away from its home-built module-signing mechanism toward the [PKCS#7](<https://tools.ietf.org/html/rfc2315>) standard. This is, Luis said, a good time to adopt the same standard for the signing of other files loaded into the kernel — firmware files in particular. 

In this plan, the enforcement of signatures on firmware would be (like enforcement of module signatures) optional; one could build a kernel without this capability. Most firmware loaded by Linux drivers is kept in the [linux-firmware repository](<https://git.kernel.org/cgit/linux/kernel/git/firmware/linux-firmware.git>); those blobs would be signed by the firmware maintainer, and would, thus, be loadable by default. To make that scheme work, Luis has proposed that the Linux Foundation create an [X.509](<https://en.wikipedia.org/wiki/X.509>) key that would be embedded in the kernel source and would be used, in turn, to sign the firmware maintainer's key. The topic of firmware for out-of-tree modules was not brought up; if nothing else, it should always be possible to add other keys to the kernel's keyring to enable loading that firmware. 

Andy Lutomirski quickly [raised some concerns](<https://lwn.net/Articles/645825/>) about how these keys would be handled. In particular, he would like to ensure that the applicability of any given key is as limited as possible. Keys used for module signing should not work for firmware signing, for example. Signatures should also spell out where a firmware blob is meant to be used in order to avoid attacks where firmware is sent to the wrong device. 

The more general question, though, as asked by [Alan Cox](<https://lwn.net/Articles/645828/>) and [Greg Kroah-Hartman](<https://lwn.net/Articles/645829/>), was: why bother signing firmware files in the first place? As Greg put it: 

I too don't understand this need to sign something that you don't really know what it is from some other company, just to send it to a separate device that is going to do whatever it wants with it if it is signed or not. 

Both asserted that, if firmware signatures are to be checked, that checking should be done by the actual device receiving the firmware. Anything else puts the kernel in the position of attesting to the validity of a firmware image that it can't know much about. 

The problem with checking signatures in the device, as David Woodhouse [pointed out](<https://lwn.net/Articles/645830/>), is that loadable firmware is often used as a way to reduce the cost of the hardware. Putting that sort of cryptographic capability into a device (one which doesn't have its operating software loaded yet, at that) would, instead, add expense and defeat the original purpose. Greg [disagreed](<https://lwn.net/Articles/645831/>) with the idea that adding signature checking would be prohibitively expensive, but the fact of the matter seems to be that hardware lacking firmware signature-checking capability is not going away anytime soon. 

David also [made the point](<https://lwn.net/Articles/645832/>) that, in the absence of an I/O memory management unit, a rogue device can do anything it wants to the running system. Thus, compromised firmware might indeed be an attractive way to attack a system. Firmware signing, he said, is a way of protecting the operating system; it is not just a service for hardware vendors. 

There is another reason for wanting this capability, though; it can be used to verify the provenance of other files loaded into the kernel as well. Luis, in particular, is concerned with the Linux "central regulatory domain agent" (CRDA) subsystem. CRDA describes the legal operating parameters for wireless network interfaces in various jurisdictions worldwide. Different countries have different rules about which frequencies can be used, maximum power levels, and more. The CRDA subsystem ensures that Linux systems play by the rules wherever they may end up. 

Luis credits CRDA with having gotten us out of the situation where manufacturers of wireless adapters refused to provide free drivers for their hardware. With CRDA in place, those manufacturers can be confident that their hardware will be operated in a compliant fashion. But that confidence is only merited if the CRDA database cannot be trivially modified by users. To that end, the database is currently signed; that signature is, on some distributions at least, verified in user space before being loaded into the kernel. 

The use of subsystem-specific cryptographic code seems like a sure path to problems at some point; getting such code right is not easy, and the number of eyeballs on, say, the CRDA signature-checking code is probably fairly low. So Luis would like to move the checking into the kernel and have it use the same code that the module loader is using. That should reduce the amount of signature-checking code in use and increase confidence that said code is actually working as planned. 

In this case, there is no device to offload the responsibility for checking a signature to; this is data that is used directly by the kernel. If the kernel is going to protect itself from a corrupted CRDA database, it needs to do the checking itself. Code that does this checking can check firmware just as easily. So it makes some sense to introduce the feature as a firmware-validation mechanism that can also check the integrity of other files loaded into the kernel. 

The end result is that this feature will probably not have too much trouble getting into the kernel once it's clear that the code works as intended. The kernel community as a whole is generally receptive to the addition of this sort of integrity-verification mechanism, as long as the policy choices remain in the hands of the user. Distributions may or may not choose to enable firmware signature checking, but the option will be there for those who want it.
