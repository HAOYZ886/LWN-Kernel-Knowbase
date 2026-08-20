---
title: The mismatched mount mess
url: https://lwn.net/Articles/762355/
date: "August 10, 2018"
category: "Filesystems-Mounting"
author: "By Jonathan Corbet August 10, 2018"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
August 10, 2018

"Mounting" a filesystem is the act of making it available somewhere in the system's directory hierarchy. But a mount operation doesn't just glue a device full of files into a specific spot in the tree; there is a whole set of parameters controlling how that filesystem is accessed that can be specified at mount time. The handling of these mount parameters is the latest obstacle to getting the [proposed new mounting API](<https://lwn.net/Articles/759499/>) into the mainline; should the new API reproduce what is arguably one of the biggest misfeatures of the current `mount()` system call? 

The [list of possible mount options](<http://man7.org/linux/man-pages/man8/mount.8.html#FILESYSTEM-INDEPENDENT_MOUNT_OPTIONS>) is quite long. Some of them, like `relatime`, control details of how the filesystem metadata is managed internally. The `dos1xfloppy` option can be used with the FAT filesystem for that all-important compatibility with DOS 1.x systems. The ext4 `bsddf` option tweaks how free space is reported in the [`statfs()`](<http://man7.org/linux/man-pages/man2/statfs.2.html>) system call. But some options can have significant security implications. For example, the `acl` and `noacl` options control whether access control lists (ACLs) are used on the filesystem; turning off ACLs by accident on the wrong filesystem risks exposing files that should not be accessible. 

It turns out that turning off ACLs by accident is indeed something that can happen on Linux systems. Eric Biederman, who has been on a bit of a crusade to force changes to the new proposed mount API, has [described how that can happen](<https://lwn.net/ml/linux-kernel/87d0uqpba5.fsf@xmission.com/>). In a simplified form, consider this set of actions: 

  * Create a large scratch file and set it up as a loopback device with [`losetup`](<http://man7.org/linux/man-pages/man8/losetup.8.html>). 
  * Create an ext4 filesystem on the device. 
  * Mount that device with the `noacl` option somewhere in the filesystem hierarchy. 
  * In another spot, mount that same filesystem with the `acl` option. 

The user who performed the second mount would naturally expect to get a filesystem with ACLs enabled — that behavior was explicitly requested, after all. But the kernel will, instead, silently apply the options used in the first mount to the second, resulting in an apparently successful mount with parameters other than those that were requested. Biederman's chief complaint is that the new API will behave in the same way; he has stated his intent to block the merging of that code until this issue is fixed. 

The source of this problem is that, in the kernel, it's really only possible to mount a filesystem once. The kernel is able to create new mount points that look like independent mounts, but it's all a single mounted filesystem underneath the cover. That means that only a single set of mount options can apply. So, as Ted Ts'o [explained](<https://lwn.net/ml/linux-kernel/20180810153902.GH21087@thunk.org/>), there aren't a whole lot of options for changing this behavior: 

So if the file system has been mounted with one set of mount options, and you want to try to mount it with a conflicting set of mount options and you don't want it to silently ignore the mount options, the *only* thing we can today is to refuse the mount and return an error. 

Some developers, including Biederman, are arguing that refusing the mount would indeed be better than ignoring the requested mount parameters. Andy Lutomirski [said](<https://lwn.net/ml/linux-kernel/CALCETrXC8Z-q+PzzqMC-McA7UdmFubVcs2dVsT0Dt+GbSqjF5A@mail.gmail.com/>) that this sort of multiple mount can go wrong in a number of ways and probably should not be allowed at all: ""It seems to me that the current approach mostly involves crossing our fingers."" There is, however, little prospect of changing how `mount()` works now, given the risk of breaking no end of administrative scripts. 

That does leave open the question of whether the _new_ API should allow this type of mount. Biederman [feels strongly](<https://lwn.net/ml/linux-kernel/87in4jwo6i.fsf@xmission.com/>) that incompatible shared mounts should be disallowed before the new API makes it into a kernel release, since it will become much harder to change afterward: 

The fact that these things happen silently and you have to be on your toes to catch them is fundamentally a bug in the API. If the mount request had simply failed people would have noticed the issues much sooner and silently b0rkend configuration would not have propagated. As such I do not believe we should propagate this misfeature from the old API into the new API. 

David Howells, the developer behind the new mount API, has [stated](<https://lwn.net/ml/linux-kernel/22361.1533913891@warthog.procyon.org.uk>) that, since the current code does not break any existing user behavior, there is no urgent need to add restrictions. But he is looking into doing so anyway, adding options so that user space can specify whether no sharing should be allowed at all, whether it should only be allowed with the same mount parameters, or whether the current behavior should apply. Others have suggested a variant on the middle case, where the mount options would just have to be "compatible" with each other, not identical. 

It turns out, though, that this limited sharing is not easy to implement either way. The core filesystem layer has no idea which mount options are compatible with each other, so there would have to be a new callback added to each filesystem implementation to answer that question. That answer doesn't just depend on the actual options; things like the security-module policy in force also have to be taken into account. It is a thorny problem, and any solution seems likely to be prone to errors. It is thus unsurprising that developers like Ts'o are [asking](<https://lwn.net/ml/linux-kernel/20180810161400.GA627@thunk.org/>) whether it is worth the effort at all. 

That is a question that has not been answered as of this writing. Assuming that Biederman doesn't back down, there will probably need to be some way of preventing shared mounts when the options are not compatible; that could come down to preventing (or at least giving an option to prevent) shared mounts entirely. Such an outcome will do little good, though, if there are enough users out there who depend on this type of shared mount. If the new API prevents them from getting their work done, they will simply stick with the old one, which will then become difficult to ever remove from the kernel. 

Biederman is right in saying that, had this particular behavior never been allowed, there would not be users who are dependent on it now. But that ship sailed a long time ago. What's left now is a mess where developers are trying to figure out what the correct behavior while avoiding causing pain to system administrators. It is a bit of a mess lacking an obvious solution.
