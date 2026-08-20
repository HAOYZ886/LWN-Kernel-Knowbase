---
title: An operation for filesystem tucking
url: https://lwn.net/Articles/927491/
date: "March 31, 2023"
category: "Filesystems-Mounting"
author: "By Jonathan Corbet March 31, 2023"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
March 31, 2023

As a general rule, the purpose behind mounting a filesystem is to make that filesystem's contents visible to the system, or at least to the mount namespace where that mount occurs. For similar reasons, it is unusual to mount one filesystem on top of another, since that would cause the contents of the over-mounted filesystem to be hidden. There are exceptions to everything, though, and that extends to mounted filesystems; [a "tucking" mechanism](<https://lwn.net/ml/linux-fsdevel/20230202-fs-move-mount-replace-v1-0-9b73026d5f10@kernel.org/>) proposed by Christian Brauner is designed to hide mounted filesystems underneath other mounts — temporarily, at least. 

Brauner's cover letter describes the intended use case in great detail; the text below is an attempt to boil that discussion down a bit. His explanation leans heavily on the concepts of shared subtrees and mount propagation; a review of [this article](<https://lwn.net/Articles/689856/>) provides some context that may make the description easier to follow. 

Filesystem tucking is aimed at systems running service and/or container workloads following the developing [image-based model](<https://lwn.net/Articles/912774/>). Each service or container within such a system has a filesystem hierarchy that is an assembly of immutable base filesystems that can have overlay filesystems mounted on top. The base filesystems provide the core operating system, while the overlays supply task-specific components, updates, and configuration files. Each container can have its own filesystem hierarchy, but often most of them will share many of the components; each container assembles its hierarchy in its own mount namespace. 

As an example, a container might mount its root filesystem from an immutable image on the host. Overlays might then supply some needed binaries or the necessary configuration files in `/etc`. Through the use of [overlayfs](<https://www.kernel.org/doc/html/latest/filesystems/overlayfs.html>), these overlays can add files to or replace specific files in the underlying image, with the result providing the needed files to the container. 

This mechanism can work well, but there are potential problems when an overlay image needs to be updated — to reflect a configuration change or apply a security update, for example. One approach would be to send a message to every container instructing it to mount the new overlay, but that can be CPU-intensive and depends on each container responding appropriately. Instead, mount propagation can be used to unmount the old overlay and mount the new one on the host; those changes will then automatically propagate into every mount namespace using that overlay. 

There is, however, a different problem with that approach: the need to unmount the old overlay before mounting the new one creates a window when the overlay is missing in the containers. That could lead to strange behavior or, possibly, an opening for a [downgrade attack](<https://en.wikipedia.org/wiki/Downgrade_attack>). It would be nice if there were a way to seamlessly move containers to the new overlay image automatically without opening that window, and without having to manage each container individually. 

The answer is filesystem tucking. Consider, for example, a simple overlay mounted on `/etc`: 

> ![\[Overlay on /etc\]](https://static.lwn.net/images/2023/tucking1.svg)

The base image provides the contents of `/etc`, in immutable form, for all containers on the system; the overlay then provides whatever additions are needed for the type of container that is running. At some point, the need to make a change to the configuration arises, so the overlay needs to be replaced. Rather than unmounting the existing overlay immediately, the first step is to "tuck" the new overlay _underneath_ the old one: 

> ![\[Tucked overlay\]](https://static.lwn.net/images/2023/tucking2.svg)

Mount propagation will cause this new overlay to be mounted in every container where it is needed, but the new overlay will not make any visible changes to the filesystem contents at this point, since the old overlay is mounted on top of it. As far as an application running inside the container is concerned, nothing has changed. But, then, the old overlay can be unmounted, yielding: 

> ![\[New overlay active\]](https://static.lwn.net/images/2023/tucking3.svg)

The old overlay is no longer masking the new one, so the new overlay becomes active. When this sequence is followed, the update to the new overlay is done atomically from the container's point of view; there is no time where the base image is directly exposed. 

It is possible to do this kind of tucking now, but not easily; the system calls for mounting (even using [the new mount API](<https://lwn.net/Articles/759499/>)) do not allow for mounting a filesystem underneath another in this way. The curious can read Brauner's cover letter for the details on how it can be done. To make life easier, Brauner proposed the addition of a new flag, `MNT_TREE_TUCK` (later changed to a separate operation called `MOVE_MOUNT_BENEATH` in [a later revision](<https://lwn.net/ml/linux-fsdevel/20230202-fs-move-mount-replace-v2-0-f53cd31d6392@kernel.org/>) of the patch set), to the `move_mount()` system call; it would cause the new mount to be placed underneath the existing mount at the mount point, rather than on top of it. There are a number of restrictions on this operation, including a prohibition on tucking a mount underneath the root filesystem, and a requirement that the caller have the privilege to unmount the filesystem under which the new one will be tucked. 

At the command-line level, Brauner describes a new option to the `mount` command to create a tucked mount. In the first posting, this option was called `--tuck`, but perhaps fearing that this would cause `mount` to join `fsck` on the list of carefully typed (and pronounced) filesystem commands, Brauner changed it to `--beneath` in the second revision. 

As of this writing, there have been no responses at all to this proposal; perhaps potential reviewers are still working their way through the cover material. Filesystem tucking seems likely to come up at the [LSFMM/BPF Summit](<https://events.linuxfoundation.org/lsfmm/>) in May. It does appear that there is a use case for this feature, though, and no immediate downsides to having it, so chances are that some form of this capability will eventually find its way into the mainline.
