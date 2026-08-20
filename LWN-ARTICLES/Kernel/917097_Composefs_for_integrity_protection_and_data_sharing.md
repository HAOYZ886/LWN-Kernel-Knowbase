---
title: Composefs for integrity protection and data sharing
url: https://lwn.net/Articles/917097/
date: "December 7, 2022"
category: "Data integrity; Filesystems-composefs"
author: "By Jake Edge December 7, 2022"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jake Edge**  
December 7, 2022

A read-only filesystem that will transparently share file data between disparate directory trees, while also providing integrity verification for the data and the directory metadata, was recently [posted as an RFC](<https://lwn.net/ml/linux-kernel/cover.1669631086.git.alexl@redhat.com/>) to the linux-kernel mailing list. [Composefs](<https://github.com/containers/composefs>) was developed by Alexander Larsson (who posted it) and Giuseppe Scrivano for use by [podman](<https://podman.io/>) containers and [OSTree](<https://ostreedev.github.io/ostree/>) (or "libostree" as it is now known) root directories, but there are likely others who want the abilities it provides. So far, there has been little response, either with feedback or complaints, but it is a small patch set (around 2K lines of code) and generally self-contained since it is a filesystem, so it would not be a surprise to see it appear in some upcoming kernel. 

#### Features

There is a lengthy introduction to composefs in the cover letter and more information in the [documentation patch](<https://lwn.net/ml/linux-kernel/8a9aefceebe42d36164f3516c173f18189f0d7e7.1669631086.git.alexl@redhat.com/>). Unlike many filesystems, composefs is not backed by a block device but instead by a set of regular read-only files: an image file that contains the directory structure and file metadata and a directory of content-addressed objects that are the file contents. Since the files themselves are in an object store, they are effectively deduplicated at the file level; files that have identical content are only stored once even if the metadata (e.g. owner, permissions, extended attributes) is different. In addition, when a file is read from composefs, the backing file is used—and cached in the page cache—so any read of another composefs file with the same content will use that page-cache entry. 

The filenames used in the object store correspond to the [fs-verity](<https://www.kernel.org/doc/html/latest/filesystems/fsverity.html>) digests of the contents, so composefs can detect any changes to files using the fs-verity mechanism. The image file can contain the digests for the files to ensure they are not changed out from under composefs and the image file itself can be protected with fs-verity as well. The fs-verity digest of the image file could be retrieved from a secure location (e.g. a signed kernel command line) and passed to the mount command, allowing composefs to ensure that the entire filesystem is unchanged from its expected state. 

The files needed for a composefs instance are created using the `mkcomposefs` tool: 

```
$ mkcomposefs --digest-store=objects rootfs/ rootfs.img
```

That command would process the `rootfs` directory, storing the image file corresponding to its directory structure and file metadata in `rootfs.img`, and create an object store with the file contents into the `objects` directory. As with Git, the `objects` directory has subdirectories named for the first two hex characters of the digest hash, each of which contains files named with the remaining characters of the hash. 

A filesystem created that way could then be mounted as follows: 

```
$ mount -t composefs rootfs.img -o basedir=objects,verity_check=2 /mnt
```

The `verity_check` option governs whether to do fs-verity checking on the image and files; 0 disables fs-verity checking, 1 only checks images that specify it, and 2 requires fs-verity. There is also a `digest` option that can be used to pass the fs-verity digest hash for the image file to the mount command, which allows composefs to verify it. That provides end-to-end verification as noted in the cover letter: ""So, given a trusted set of mount options (say unlocked from TPM), we have a fully verified filesystem tree mounted, with opportunistic fine-grained sharing of identical files."" 

#### Use cases

Multiple containers on a system often share many files, but that file data is typically replicated for each container image. Composefs can be used to create multiple, different, read-only container image files that all refer to the same object store, so that all of the files are only stored once. In addition, files that are being used by multiple containers simultaneously (e.g. shared libraries) will likely reside in the page cache, saving a slower retrieval from persistent storage. 

The second use that Larsson and Scrivano envision is to [replace the "link farm"](<https://blogs.gnome.org/alexl/2022/06/02/using-composefs-in-ostree/>) that gets created for OSTree-based root filesystems with a composefs mount. Currently, OSTree uses hard links into its content-addressed object store, but that structure is not protected against changes because it does not get validated at run time, only when it is being created. By using fs-verity on the image file and object store, though, a composefs-based root filesystem would have its integrity protected automatically. 

Beyond those two immediate uses for composefs, there are others on the horizon: 

> [...] there seems to be a wealth of other possible uses. For example, many systems use loopback mounts for images (like lxc or snap), and these could take advantage of the opportunistic sharing. We've also talked about using fuse [[Filesystem in Userspace](<https://en.wikipedia.org/wiki/Filesystem_in_Userspace>)] to implement a local cache for the backing files. I.e. you would have a second basedir be a fuse filesystem, and on lookup failure in the first basedir the fuse one triggers a download which is also saved in the first dir for later lookups. There are many interesting possibilities here. 

The reaction to the patches has been non-existent so far, other than some [documentation updates](<https://lwn.net/ml/linux-kernel/Y4YSYcqv3LdbseVy@debian.me/>) suggested by Bagas Sanjaya. The facilities provided by composefs seem useful, however, and build on the fs-verity feature that is, after [some fits and starts](<https://lwn.net/Articles/790185/>), already present in the kernel. There is also work in progress to [support composefs in OSTree](<https://github.com/ostreedev/ostree/pull/2640>), but there is something of a chicken-and-egg problem until the filesystem lands in the kernel. With luck, and a continued lack of any serious opposition to composefs, that problem could be addressed—perhaps even early next year.
