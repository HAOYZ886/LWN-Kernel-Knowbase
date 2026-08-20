---
title: "Quickly: Filesystems and containers / Self-encrypting drives"
url: https://lwn.net/Articles/684832/
date: "April 27, 2016"
category: "Block layer; Containers; Virtualization-Containers"
author: "By Jake Edge April 27, 2016 LSFMM 2016"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
April 27, 2016

* * *

[LSFMM 2016](<https://lwn.net/Articles/lsfmm2016/>)

Two lightning talks ended day one of the 2016 Linux Storage, Filesystem, and Memory-Management Summit. One looked at the problems with user namespaces and the image files used by unprivileged containers. The other was concerned with [self-encrypting disk drives](<https://en.wikipedia.org/wiki/Hardware-based_full_disk_encryption>). 

#### Containers and filesystem images

James Bottomley kicked things off by describing a problem inside containers using user namespaces, where root in the container is mapped to some unprivileged user outside the namespace. Filesystems that are mounted in the container will have files that are owned by root but, by the time a read or write hits the virtual filesystem (VFS) layer, the UID is for the unprivileged user, so those operations fail. There is a need to not do this UID remapping for some mounts. 

One way to do that would be to give up on using bind mounts and to use FUSE mounts that are user-namespace-aware instead. David Howells said that the performance would be lacking for FUSE filesystems, but Bottomley was not so sure. There have been lots of performance enhancements to FUSE, so "in theory we can get reasonable performance". 

As he saw it, there were three options: use a FUSE filesystem, re-work the user-namespace remapping code throughout the VFS, or take the patches that systemd uses. He expected that Al Viro (who was not present for the talk) was likely to be resistant to making these changes at the VFS level. Alternatively, there are twelve or so "horrible patches" that systemd uses to handle this problem, but he noted that there are many more users of unprivileged containers that also need a solution to the problem—it is not systemd-specific at all. 

There was a question about why the filesystem image wasn't simply changed to remap the UIDs before mounting. Bottomley said that breaks the checksum of the image, which is used to verify its integrity. 

Ted Ts'o suggested that a specific UID-remapping filesystem could be created, along the lines of overlayfs. That would limit the UID remapping to that filesystem, rather than scattering it throughout the VFS layer. 

That idea had some appeal. Bottomley noted that FUSE has options for better performance, including direct I/O and writeback caching. But it would seem that the overlayfs-based solution may be given a long look. 

#### Self-encrypting drives

Keith Busch wanted to discuss self-encrypting drives and how best to support them in Linux. In particular, what is responsible for unlocking the drives after the system goes to sleep? When the power is removed from these drives, they lock; when power is restored, they require user input to unlock them. Other operating systems store the user's password somewhere (such as in EFI variables) and then play it back when the system wakes to unlock the drives. 

Martin Petersen questioned the value of self-encrypting drives other than as a check-mark for "security". Overall, there was general skepticism about the security value of the feature. Busch said that there were customer requests to support the feature, however.. 

There are [Trusted Computing Group](<http://www.trustedcomputinggroup.org/>) (TCG) specifications to handle the authentication problem, but Busch guessed that adding that code to the kernel would not be welcome. Hannes Reinecke concurred, saying that the kernel security developers would not want that since the TCG code implementation is "horrible". 

Dan Williams suggested that the BIOS could put up a prompt to ask for the password to unlock the drive. That could be done as a pre-resume hook that re-runs the authentication step. Others, though, believed the problem had already been solved: "dm-crypt, check it out". In the end, there did not seem to be much support for handling these devices, even though both Busch and Brian King said that there are growing customer requests.
