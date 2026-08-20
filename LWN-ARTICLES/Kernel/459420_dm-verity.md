---
title: "dm-verity"
url: https://lwn.net/Articles/459420/
date: "September 19, 2011"
category: "Device mapper; Security-Integrity verification"
author: "By Jonathan Corbet September 19, 2011"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
September 19, 2011

It is not often that Netflix employees show up on linux-kernel to advocate for the merging of specific patches. But that is exactly what has happened after a posting of a new device mapper module called dm-verity. As one might expect, dm-verity has little to do with, say, efficient sorting of DVD mailings. It is, instead, a classic piece of security technology with the potential to work in the user's interests - or against those interests. 

The purpose of dm-verity is to implement a device mapper target capable of validating the data blocks contained in a filesystem against a list of cryptographic hash values. If the hash for a specific block does not come out as expected, the module assumes that the device has been tampered with and causes the access attempt to fail. It has been put forward by Mandeep Singh Baines of Google's Chromium OS team, but there appears to be interest in this capability beyond that small group. 

At the core of this new facility is a module called dm-bht, which works with a list of block numbers and their associated hash values. This list is organized into a simple tree for quick access to the hashes for arbitrary blocks. In essence, the leaves of the tree are pages containing hash values; each higher level in the tree contains hashes of the blocks below it. Verifying a block requires not only checking the hash value for that specific block; it is also necessary to verify hashes up to the root of the tree. If the hash for the tree root (which is assumed to be trusted) checks out, all is well. The dm-bht code can use any hash algorithm supported by the kernel's crypto API; SHA1 is given as an example, but others can be used as well. 

dm-verity implements a read-only target; it is assumed that there is no need to change the data protected by this scheme (being, most likely, the binaries run by the system itself) during operation. The tree of block hashes is stored with (or near) the data itself, but the root hash must be passed in externally. If that root hash comes from a trusted source, it should be possible to detect any modification of the disk, in either the data itself or in the stored hash tree. So, if all goes well, a system running with dm-verity can be assured that the underlying software has not been changed. It's worth noting that integrity checking for any specific block does not happen until the kernel tries to read that block into the page cache. There is, thus, no need for a lengthy verification process at boot or mount time. 

All of this depends on getting the right hash value into the system at startup time. If some sort of hardware-verified trusted bootloader is in use, that can probably be done in some sort of secure manner. Device mapper setup is a complex task requiring some sort of running user space. This means that a system using dm-verity will need some other mechanism to load a trusted kernel and initramfs or the whole chain will break. A trusted bootloader can achieve that kind of setup; another example given by the developers is a system booting from a "known good" source like a USB stick that is never left unattended. 

One might wonder how dm-verity differs from existing features like the [extended verification module](<https://lwn.net/Articles/394170/>). EVM requires and uses a trusted platform module (TPM) on the system to be verified; as long as the initial boot step can be secured, dm-verity is able to work without a TPM. It also seems likely that dm-verity will be faster since it does on-demand verification of blocks; there is no need to verify entire files before the first block can be accessed. 

Wesley Miaw of Netflix [made it clear](<https://lwn.net/Articles/459422/>) that this patch is seen with favor there: 

Netflix would like dm-verity to be included in the Linux kernel. Over the past year, we have been working with Google and porting dm-verity onto a number of consumer electronics devices running embedded Linux. Demand for this feature has been high and we see a lot of benefit associated with making dm-verity part of the official kernel. 

The reasons for this interest should be fairly clear: dm-verity will make it easier to create locked-down Linux-based systems that will enforce whatever DRM requirements the movie studios may see fit to impose. Thanks to dm-verity, there will no longer be pirated films circulating on the Internet; or, perhaps, that's the sort of outcome that only happens in the movies. Whether or not the effort is futile, it shows that tools like dm-verity can be used to harden Linux-based systems in ways that are hostile to their users. 

To an extent, Google's interests may align with those of Netflix: Chromebooks that can stream content from Netflix will be more attractive than those that cannot. But dm-verity also fits the ChromeOS concepts of minimal, trustworthy devices with data stored on Google's servers. For users who like this mode of operation, this kind of built-in integrity protection is a positive feature. Google can, one hopes, be trusted to hold the user's data; a suitably verified device can be trusted not to leak that data or the user's credentials to an attacker. Even if the running system is compromised through some sort of malware attack, a simple reboot should either put things right or make it clear that the machine can no longer be trusted. 

As long as this functionality is under the user's control, it can be made to serve the user's interests. The "developer mode switch" designed into Chromebooks seems like a good compromise in this area. Some vendors will, beyond doubt, choose to incorporate tools like dm-verity without giving owners the ability to turn it off. That is not a good thing, but neither is it anything new.
