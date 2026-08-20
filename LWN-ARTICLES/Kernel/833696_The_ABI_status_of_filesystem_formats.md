---
title: The ABI status of filesystem formats
url: https://lwn.net/Articles/833696/
date: "October 8, 2020"
category: "Development model-User-space ABI; Filesystems-ext4"
author: "By Jonathan Corbet October 8, 2020"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
October 8, 2020

One of the key rules of Linux kernel development is that the ABI between the kernel and user space cannot be broken; any change that breaks previously working programs will, outside of exceptional circumstances, be reverted. The rule seems clear, but there are ambiguities when it comes to determining just what constitutes the kernel ABI; tracepoints are [a perennial example](<https://lwn.net/Articles/799262/#tp>) of this. A recent discussion has brought another one of those ambiguities to light: the on-disk format of Linux filesystems. 

Users reporting kernel regressions will receive a varying amount of sympathy, depending on where the regression is. For normal user-space programs using the system-call API, that sympathy is nearly absolute, and changes that break things will need to be redone. This view of the ABI also extends to the virtual filesystems, such as `/proc` and sysfs, exported by the kernel. Changes that break things are deemed a little more tolerable when they apply to low-level administrative tools; if there is only one program that is known to use an interface, and that program has been updated, the change may be allowed. On the other hand, nobody will be concerned about changes that break out-of-tree kernel modules; the interface they used is considered to be internal to the kernel and not subject to any stability guarantee. 

But those are not the only places where user space interfaces with the kernel. Consider, for example, [this regression report](<https://lwn.net/ml/linux-kernel/20201005081454.GA493107@localhost/>) from Josh Triplett. It seems that [an ext4 filesystem bug fix](<https://git.kernel.org/linus/e7bfb5c9bb3d>) merged for 5.9-rc2 breaks the mounting of some ext4 filesystems that he works with. 

These filesystems are created for some unspecified internal purpose with an unreleased internal tool. They are meant to be read-only, so there will never be any need (or ability) to create new files on them. As a space-saving measure, this tool overlays all of the block and inode bitmaps onto a single block, set to all ones, indicating that all blocks and inodes are already allocated. To indicate that this has been done, this tool marks the filesystem with a special flag (`EXT4_FEATURE_RO_COMPAT_SHARED_BLOCKS`). This flag is defined by the ext4 tools, but is not used by the kernel in any way. It is placed in the set of read-only compatibility flags, though, meaning that a kernel that sees it in a filesystem will know that said filesystem can be safely mounted, but only in read-only mode. 

Until 5.9-rc2, mainline kernels were happy to mount these filesystems. The commit highlighted by Triplett changed that situation by adding some checks to the mount-time ext4 verifier that ensures that the filesystem image is valid. As a result of these new checks, the overlapping bitmaps are detected, the kernel complains, and any attempt to mount the filesystem fails. Something that used to work no longer does — the definition of a regression. Triplett included in his report a small patch that disables the validity check when the `EXT4_FEATURE_RO_COMPAT_SHARED_BLOCKS` flag is present, rendering his filesystems mountable again. 

That change is likely to be merged, but it has not brought great joy to the filesystem developers, who see it as a sign of things going wrong. XFS maintainer Darrick Wong [argued](<https://lwn.net/ml/linux-kernel/20201006025110.GJ49559@magnolia/>) that "unofficial" filesystem variants should not be supported: 

I disagree; creating undocumented forks of an existing ondisk format (especially one that presents as inconsistent metadata to regular tools) is creating a ton of pain for future users and maintainers when the incompat forks collide with the canonical implementation(s). 

Triplett [responded](<https://lwn.net/ml/linux-kernel/20201006050306.GA8098@localhost/>) that filesystem images should fall within the realm of the kernel ABI: 

I was generally under the impression that mounting existing root filesystems fell under the scope of the kernel<->userspace or kernel<->existing-system boundary, as defined by what the kernel accepts and existing userspace has used successfully, and that upgrading the kernel should work with existing userspace and systems. If there's some other rule that applies for filesystems, I'm not aware of that. 

Ext4 maintainer Ted Ts'o [said](<https://lwn.net/ml/linux-kernel/20201006133533.GC5797@mit.edu/>) that he was not opposed to Triplett's patch, but suggested that further patches to support this tool might receive a chillier reception. He then told the story of the `make_ext4fs` tool, an independently written ext4 filesystem creator used for years by Android. It created a long list of compatibility and corruption problems, he said, that took years to iron out. Third-party filesystem tools are prone to such problems, he said: 

As far as I'm concerned, it's not just about on-disk file system format, it's also about the official user space tools. If you create a file system which the kernel is happy with, but which wasn't created using the official user space tools, file systems are so full of state and permutations of how things should be done that the opportunities for mischief are huge. 

Filesystem developers have a certain natural aversion to "mischief", so it is unsurprising that Ts'o would prefer that these outside tools simply not exist. At a minimum, he suggested, future policy should say that filesystem-image regressions would only need to be addressed for images that were created and managed by the designated official tools. He requested that Triplett find a way to get rid of his custom tool. 

Triplett [disagreed](<https://lwn.net/ml/linux-kernel/20201007080304.GB1112@localhost/>) with that policy suggestion, saying that a more reasonable approach would be that any filesystem images that pass the `e2fsck` checker should be supported. Not all tools that work with the ext4 format should have to live in the e2fsprogs repository; to say otherwise, he added, would be tantamount to saying that the FreeBSD kernel, which has an ext4 filesystem driver, should live in e2fsprogs too. The tool in question here, which Triplett finally [described](<https://lwn.net/ml/linux-kernel/20201007201424.GB15049@localhost/>) late in the conversation, appears equally unsuited to inclusion in the e2fsprogs repository. 

Triplett concluded that message with a restatement of his request: 

The *only* thing I'm asking, here, is "don't break things that worked". And after this particular item, I'd be happy to narrow that to "don't break things that e2fsck was previously happy with". 

[Ts'o's response](<https://lwn.net/ml/linux-kernel/20201008021017.GD235506@mit.edu/>) made it clear that he is uninclined to grant that wish; that would, he said, make it impossible to fix security-related problems related to invalid filesystem images, of which there are many. Many of these invalid images — often generated by fuzzing tools or attackers — pass `e2fsck` until the problems are found and fixed. Grandfathering in any image that passes `e2fsck` would thus, he said, require invalid filesystems to be supported forever. He concluded with some suggestions for other ways to solve Triplett's problem and requested that Triplett work more closely with the ext4 developers in the future. 

As of this writing, that is where the discussion has stopped. Ts'o's willingness to apply the fix for the immediate problem means that there is no pressing need to resolve the larger issue of regressions involving filesystems; that, in turn, means that the issue is likely to come up again at some point. The kernel ABI is a large and amorphous thing, and many of the boundaries are fuzzy at best. Filesystems are one area where that boundary has not yet been fully explored; somebody is likely to inadvertently end up on the wrong side of it sooner or later.
