---
title: An overlayfs update
url: https://lwn.net/Articles/1077052/
date: "June 12, 2026"
category: Overlayfs
author: "By Jake Edge June 12, 2026 LSFMM+BPF"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
June 12, 2026

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2026/>)

In a shortened session in the filesystem track at the 2026 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>), Amir Goldstein gave an update on the [overlayfs union filesystem](<https://docs.kernel.org/filesystems/overlayfs.html>). There are some new features over the last few years that he wanted to mention, along with looking at the status of nesting overlayfs layers. The [composefs use case](<https://lwn.net/Articles/922851/>) that was [discussed at the summit in 2023](<https://lwn.net/Articles/933616/>) has led to some interesting changes to overlayfs. 

Overlayfs provides a way to create a single mounted filesystem that is created from multiple other filesystems fused together. It presents a union of the files in the various filesystems, though the underlying filesystems are ordered so that entries from filesystems above take precedence over the same file and directory names in the lower layers. Often, the top layer is writable so that users can change the files as they appear in the mounted overlayfs without actually changing anything in the (typically read-only) lower layers. 

Goldstein began by noting that Miklos Szeredi, who developed overlayfs and co-maintains it with him, once talked about a time "when overlayfs is done", which reminded him of the concept of "[the end of history](<https://en.wikipedia.org/wiki/End_of_history>)". ""So, what happened since the end of history?"", Goldstein asked with a grin. For one thing, Szeredi added the ability to mount overlayfs filesystems in user namespaces ""after he thought he was done with it"". 

Supporting composefs, where the file contents are stored as content-addressable objects using [fs-verity](<https://www.kernel.org/doc/html/latest/filesystems/fsverity.html>) for integrity protection, has led to some new concepts for overlayfs. There can now be [data-only lower layers](<https://docs.kernel.org/filesystems/overlayfs.html#data-only-lower-layers>), lacking metadata, which are somewhat similar to the [metacopy feature](<https://docs.kernel.org/filesystems/overlayfs.html#metadata-only-copy-up>) that was already present, but allows verifying the connection between the metadata of the inode and its contents using fs-verity. 

He noted that Christian Brauner had switched overlayfs to use [the "new" mount API](<https://lwn.net/Articles/759499/>), which lifted some restrictions on things like the number of lower layers and the path-name length for them. Brauner described another change that would allow separating the credentials needed to mount a layer as part of an overlayfs from those needed to access the layer after the mount. For example, the mounter may require a different SELinux context than the one that will be used by the tasks accessing the mount, he said. The feature has more widespread applicability, but for overlayfs, it allows the administrator to specify the credentials that will be used for accessing the layer via the mounted filesystem. 

Goldstein said that, previously, overlayfs had a single set of credentials, which corresponded to the task mounting the filesystem. It used those credentials to access the lower layers, rather than those of the user who was doing the filesystem operations on the mounted overlayfs. Now the credentials used to mount the filesystem can be separated from those used when the filesystem is accessed. ""It's a good sign for our security model that we need two different people to provide two different explanations"", Brauner said with a laugh. ""None of which would be understood"", Goldstein added to general laughter. 

#### Nested overlayfs

Overlayfs can be nested, which means that one of the layers making up an overlayfs filesystem is, itself, an overlayfs filesystem, Goldstein said. It has been possible to do that for more than ten years. The classic example is to create an overlayfs with two, say, XFS layers and use that as the lower layer; the upper layer is some other filesystem and all are fused into an overlayfs with the first overlayfs as its lower layer. ""I'm not sure exactly"" when that is useful, he said, but sometimes in containers, or for [OpenWrt](<https://openwrt.org/>), the root filesystem is an overlayfs and users want to be able to make non-destructive changes to it. 

There are other nested-overlayfs types, including one for composefs as he had mentioned earlier. Composefs knows the overlayfs on-disk format and how to create the extended attributes needed for that structure, so it can create its own lower layers to contain its data. Features were added to overlayfs in the 6.7 kernel to allow composefs to create the needed extended attributes, which are normally treated as private, overlayfs-only attributes. But, all of the nested types would only allow overlayfs as a lower layer, not for the upper. 

Another use case that had come up recently was running a Docker application inside another container that had an overlayfs root filesystem. Docker will try to create an overlayfs using the existing root filesystem as the upper layer, but an overlayfs cannot be the upper layer. Instead, it unpacks its image using the "naive storage driver", which copies all of the files out of their layers and into a flat filesystem, which takes time and lots of I/O. It would be nicer if overlayfs could be extended so that it could use another overlayfs as its upper layer too. ""That would be the end of nested overlayfs history"", he said with a grin. 

Brauner asked about whether overlayfs would allow multiple lower layers that were themselves overlayfs, but Goldstein was not sure there was a use case for it. The number of nested layers that can be used in an overlayfs is capped in the kernel based on stack-depth concerns; it could increased, but only if it could be shown that it would not overflow the kernel stack. 

Instead of nesting the overlayfs layers, it might make sense to collapse them, Brauner said. Overlayfs could collect the credentials needed for accessing the different layers, but effectively treat them as a single layer so that there are no nesting limits due to stack concerns. Szeredi wondered how that would work; there was some discussion between he and Goldstein before Brauner described what he was envisioning. 

Traditionally, users have updated their systems using package managers, but much of the industry is moving toward image-based updates, Brauner said. Systemd supports this by having a read-only `/usr` that gets updated with multiple system extensions (i.e. [sysext](<https://www.freedesktop.org/software/systemd/man/latest/systemd-sysext.html>)) layers added on top, all of which get fused into a single overlayfs. Over time, there are more and more sysext layers, so systemd has to reassemble the overlayfs periodically and then swap in the new one in place of the old. It would be nicer if it could just operate on the overlayfs directly to see what the layers are and to swap in new ones as needed. 

Lennart Poettering said that the systemd developers would really like a tool of some kind to be able to see the different layers that make up the `/usr` mount. That would allow them to cryptographically trace a file back to its origin in the fs-verity-protected data. Goldstein said that providing some kind of introspection API was doable; it is mostly a matter of iteratively working out the details of the API. 

Ted Ts'o wondered if the use case being described was similar to doing distribution updates in an overlayfs-based layered installation (e.g. with base and package layers); the user may change a configuration file that now needs to be updated, which leads to some kind of conflict-resolution process. Brauner said that it was a completely different use case. In this one, the `/usr` filesystem is read-only and, ideally, `/etc` will be also. If the user is ever shown a diff and has to make a choice, ""you've already lost the plot in my opinion"". 

For the `/etc` case, the lower layers would be read-only, with a writable layer at the top so that users can make changes to the configuration, an attendee said. Sometimes the user wants to remove their changes and revert to the version in the lower layers, but deleting their file leads to a [whiteout](<https://docs.kernel.org/filesystems/overlayfs.html#whiteouts-and-opaque-directories>); he would like to have a way to reveal the underlying file instead of blocking it with the whiteout. Brauner asked if he wanted that on an individual-file basis or for the whole filesystem; the former requires some system-call-level change, which is probably harder. 

Goldstein asked what was known about the file to reveal; is it just the file name or is there a [file handle](<https://www.man7.org/linux/man-pages/man3/handle.3.html>)? That latter might be more useful because there are already guards in overlayfs to prevent using a file handle to evade the whiteout; those could perhaps be overridden under some circumstances. David Howells was concerned about nested overlayfs and identifying which file should be resurrected, but Goldstein said that the overlayfs file handle does describe both layers, so it could be used. But, ""I'm not committing to this"", he said. The session ran out of time shortly thereafter.
