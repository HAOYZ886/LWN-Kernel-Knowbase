---
title: A decision on composefs
url: https://lwn.net/Articles/933616/
date: "June 7, 2023"
category: "Filesystems-composefs"
author: "By Jake Edge June 7, 2023 LSFMM+BPF"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jake Edge**  
June 7, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

At the end of our [February article about the debate](<https://lwn.net/Articles/922851/>) around [the composefs read-only, integrity-protected filesystem](<https://lwn.net/Articles/917097/>), it was predicted that the topic would come up at the [2023 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023>). That happened on the second day of the summit when Alexander Larsson led a session on composefs. While the mailing-list discussion was somewhat contentious, the session was less so, since [overlayfs](<https://docs.kernel.org/filesystems/overlayfs.html>) can be made to fit the needs of the composefs use cases. It turns out that an entirely new filesystem is not really needed. 

Larsson began by looking at the use case that spurred the creation of composefs. At Red Hat, image-based Linux systems are created using [OSTree/libostree](<https://ostreedev.github.io/ostree/>); they are not the typical physical block-device images, however, as they are more like "virtual images". There is a content-addressed store (CAS) that contains all of the file content for all of the images. In order to build a directory hierarchy for the virtual image, a branch gets checked out from the OSTree repository, which contains the metadata and directory information for the image; OSTree then builds the directory structure using hard links to the CAS entities. 

[ ![\[Alexander Larsson\]](https://static.lwn.net/images/2023/lsfmb-larsson-sm.png) ](<https://lwn.net/Articles/933960/>)

A system created this way "is very lean", because it is flexible and easy to update, so the Red Hat developers want to use OSTree-based images for containers and other types of systems. But there is a missing feature that they would like to have: some kind of tampering prevention as with [dm-verity](<https://docs.kernel.org/admin-guide/device-mapper/verity.html>). There are two main reasons that a tamper-proof filesystem is desired: security, to provide a trusted boot, for example, and safety, such as protecting the data used by a self-driving car. [Fs-verity](<https://www.kernel.org/doc/html/latest/filesystems/fsverity.html>) provides much of what they are looking for, but it does not go far enough; it is concerned only with protecting the file contents, not the file metadata or the directory structure. 

So, back in November, he and Giuseppe Scrivano [posted composefs](<https://lwn.net/ml/linux-kernel/cover.1669631086.git.alexl@redhat.com/>), which is like a combination of a read-only filesystem, such as [SquashFS](<https://docs.kernel.org/filesystems/squashfs.html>) or [EROFS](<https://docs.kernel.org/filesystems/erofs.html>), with overlayfs. Composefs just contains the metadata and the directory structure; it gets mounted as the equivalent of the overlayfs upper layer, with the CAS as the lower layer. So referencing a file by its name actually resolves to the entry in the CAS. 

If all of the files in the CAS have fs-verity enabled for them, those digest values can be used in the creation of the image for the composefs metadata, which itself is protected with fs-verity. When composefs is mounted, the expected digest for the metadata image is passed to the `mount` command, so that it can be verified; the Merkle tree of digests for the CAS is part of that image, so everything is protected against any kind of change (be it malicious or cosmic-ray induced). 

In the "sometimes heated discussions about this", it turns out that there are already features in overlayfs "that sort of make this possible". Files can have extended attributes (xattrs) that specify that the metadata is separate from the file data (`overlay.metacopy`) and that the names are different in the layers (`overlay.redirect`). The idea would be to create an overlayfs with two filesystems: the CAS is the lowest layer and a read-only EROFS loopback image would be above that with its xattrs pointing into the CAS. 

There are some missing features, though. In order to support fs-verity on the file contents, there needs to be a `overlay.verity` xattr to tell overlayfs to verify the file contents based on the digest in the xattr. There also needs to be a mount option to specify that every file must have a digest. There are pending patches to add those features to overlayfs. 

There were some performance measurements ("of dubious quality") that were done using `ls -lR` which showed some lookup amplification in overlayfs. For no real reason, overlayfs was looking up the underlying CAS file; Amir Goldstein called overlayfs "too eager" and has posted patches to support lazy lookups, so that the lower-layer file is not looked up until it is actually needed. An overlay filesystem is a union filesystem, which combines the entries of all of the layers, but that is not needed for this use case, Larsson said. You could use [overlay whiteouts](<https://docs.kernel.org/filesystems/overlayfs.html#whiteouts-and-opaque-directories>) to hide the underlying CAS files, but Goldstein's lazy-lookup patches also add data-only layers, which do not have the file names from their filesystems visible in the combined overlayfs. 

The basic question for the room, Larsson said, was what the approach should be to getting something upstream to solve his (and others') use cases. "This where the talk gets kind of short because I think most people are leaning toward using the existing code in overlayfs", rather than add a new filesystem. It is less code to maintain, which is always beneficial, but also the features that would be added to overlayfs are useful in their own right. 

There are a few cons, but "I think they're pretty minor"; he does not like loopback mounts because they are global and are not namespaced. In addition, the performance of the overlayfs version is roughly the same as composefs (after the lazy lookup is added), but having two filesystems does double the number of inodes and directory entries (dentries) that are in use. He wondered if anyone thought that a custom filesystem was the right approach. 

If you ask a room full of kernel developers if a new filesystem is needed, "the answer is almost always going to be 'no'", Josef Bacik said to laughter; "you made our argument for us" since an existing filesystem can cover the needed features. Larsson agreed that he would rather not have to maintain a filesystem; "I have enough code to maintain". 

Larsson was asked about the problems with loopback mounts. He said that there are people working on solutions, but a container must have a loopback device available. The loopback device is global, however, so it can see all of the loopback mounts in the entire system. Christian Brauner said that he has working patches for doing proper namespacing for global system devices like loopback; there are iSCSI people who are interested in it as well. He hopes that it is just a matter of time before that problem is solved. 

There are two facets to the problem of global devices; they appear as device nodes in the `/dev` tree but are also sysfs entries, Brauner said. He did not want to do a "half-assed namespacing" where he only dealt with the device nodes and did not handle the sysfs entries, but he had to step carefully in order to avoid breaking backward compatibility. 

Other mechanisms for handling the integrity and/or trust level of container images were discussed, some of which overlapped the [FUSE passthrough](<https://lwn.net/Articles/932060/>) discussion the previous day or the session on mounting filesystems in user namespaces coming later in the day. Allowing unprivileged users to mount random images from, say, DockerHub, is not something that will ever be supported, Brauner said. Goldstein agreed, noting that something like a BPF verifier for filesystem images would be needed to ensure that they would not crash the kernel. James Bottomley thought there was a class of simple filesystem images that the kernel could verify before mounting, even for unprivileged users. But Bottomley's idea was not entirely well-received in the room.
