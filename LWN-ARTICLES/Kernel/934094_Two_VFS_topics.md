---
title: Two VFS topics
url: https://lwn.net/Articles/934094/
date: "June 9, 2023"
category: "Filesystems-Virtual filesystem layer"
author: "By Jake Edge June 9, 2023 LSFMM+BPF"
---

By **Jake Edge**  
June 9, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

Two different topics concerning the virtual filesystem (VFS) layer were the subject of a session led by VFS co-maintainer Christian Brauner at the [2023 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023>). As might be guessed, it was a filesystem-track session; Brauner had three separate items he planned on bringing up, but the discussion on the first two consumed the whole half-hour—and then some. A mechanism to avoid media-change races when mounting loop (or loopback) and other devices was disposed of fairly quickly, but the discussion around the mount-beneath feature went on at length. 

#### Diskseq

There is an issue for container runtimes, Brauner began, because they use a lot of loop devices, but the backing file for a device can change without notification to the runtime. It is a longstanding problem, but Christoph Hellwig came up with the idea of providing a monotonically increasing disk sequence number, which can be used to detect media changes. It would also be useful to detect that a USB storage device has been removed and a new one inserted. 

The value of the sequence number would be queried using the `BLKGETDISKSEQ` `ioctl()` command, which means that user-space processes could detect when loop (and other) devices have changed their media. There would also be entries in a new `/dev/disk/by-diskseq/` directory so that disks can be referenced by their sequence number. This "eliminates a bunch of races, but not all of them", Brauner said. 

[ ![\[Christian Brauner\]](https://static.lwn.net/images/2023/lsfmb-brauner-sm.png) ](<https://lwn.net/Articles/934249/>)

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

There is a kind of a time-of-check-to-time-of-use race that can still occur, which could lead to the wrong media getting mounted. He has pitched an idea to Hellwig to add a "source-diskseq" property to `fsconfig()`; before the actual mount occurs, the source and the source-diskseq properties could both be verified for the block device, ensuring that only the proper media is actually mounted. 

In general, Brauner thinks that these changes are uncontroversial, but he wanted to run them by attendees. One implication of the changes is that block-backed filesystems that have not yet been switched over to the [new mount API](<https://lwn.net/Articles/759499/>) will need to be. He is fine with doing the work to make that happen. Once it is done, there is still a bit of work that needs to be done on the block layer to support the sequence numbers. 

Josef Bacik asked how many filesystems still needed to be converted ("other than one specific filesystem that we know about"). Brauner said that he was not sure, he just assumed that some still exist. Bacik said that he would be doing the conversion for Btrfs "next week, honest", which was met with a good bit of laughter. Brauner said he could probably do it, if needed, but Bacik said he had multiple requests for switching Btrfs, so he would be getting to it soon. 

Lennart Poettering suggested that mounted filesystems should also be checking the sequence number to ensure that something surprising has not happened underneath them. Ted Ts'o said that Hellwig has sent patches that would provide a mechanism for the block layer to inform a filesystem that the media has changed, so that the filesystem can simply shut down. It is better if the filesystem is informed about an eject (i.e. media-removed) event, rather than having to check frequently to see if the sequence number has changed. 

#### Mount beneath

The second topic that Brauner wanted to discuss was the mount-beneath (formerly [filesystem tucking](<https://lwn.net/Articles/927491/>)) operation. It is a way to upgrade or replace a mount by mounting a new filesystem beneath the one being replaced in the mount stack, so that the underlying mount point is not exposed in the process. There is a tricky requirement, however, he said, in that it needs to mesh well with [mount propagation](<https://www.kernel.org/doc/html/latest/filesystems/sharedsubtree.html>). 

One use case is for containers with a shared `/usr` that gets periodically updated. Without the new feature, each update of `/usr` means that the new one gets mounted atop the existing stack of previous versions; a system with 1000 containers and five updates has 5000 mount entries. Alternatively, unmounting the old `/usr` first leaves a window in which the underlying mount point is exposed to the services in the container. 

The mount-beneath feature was the easiest way that he came up with to avoid those problems when updating filesystems. The kernel walks the mount stack for a given mount point to find the topmost mount and then it inserts the new mount just below the topmost. Then the topmost mount can be unmounted and users will never see any lower mounts or the mount point; they either see the topmost mount or the new mount. It is a way to replace a mount without falling into all of the complexities that would come from actually directly doing a mount-replace operation, he said. 

Ts'o asked how beneath mounts interacted with [overlayfs](<https://docs.kernel.org/filesystems/overlayfs.html>). Brauner said that there should be no problems with that because the overlayfs mount includes all of its constituent filesystems into a single mount. So a new overlayfs can be mounted beneath an existing one without any difficulty. 

Brauner demonstrated the feature, which is a bit hard to describe; those interested may want to see it in the [YouTube video of the session](<https://www.youtube.com/watch?v=LnLOrJb02yc>) (right around 14:08). He wanted to show that when mounting multiple times, there is an amplification effect due to mount propagation. If the parent mount and the child mount are in the same peer group, they propagate to each other. Stacking a bunch of identical mounts on top each other sets up something approaching a combinatorial explosion of mounts due to mount propagation. He is unsure if these semantics were intended, but wanted to avoid that for beneath mounts. 

David Howells asked if it would be easier to have a "swap mount" operation that would switch an existing mount for a new one. Brauner said that is effectively the same as the replace operation he had already mentioned. If a mount that is being replaced has child mounts (on subdirectories), they would need to be moved to the replacement, which may or may not have the right child mount points. Mount beneath neatly sidesteps that problem by leaving the problem of what to do with the child mounts up to user space; before it can unmount the old filesystem, it will need to do something with the child mounts. 

Howells was concerned that inserting a mount into the stack, as proposed for mount beneath, would cause problems, but Brauner said that can already happen today. There are ways to insert a mount beneath another. Since that does not cause a problem today, he believes that it will not be one for beneath mounts. He demonstrated some of that around 19:30 in the video as well. 

Remote participant Al Viro pointed out that a child mount on, say, `/usr/local` in the old to-be-updated filesystem could get lost when using mount beneath. Once a new `/usr` was inserted below, unmounting the old `/usr` will only succeed if it is a lazy unmount but then the old `/usr/local` is no longer accessible. It is inconvenient to have to mount `local` (and any other mounts on `/usr` subdirectories) on the new `/usr` before doing the mount beneath operation, but that is what has to be done to preserve the hierarchy. Brauner agreed that was the case, but he does not see it as a big problem. 

Viro said that the new `/usr` could be mounted somewhere accessible, each of the subdirectory mounts on the existing `/usr` could then be bind-mounted to the new in the right places. After that, the new one could be mounted beneath the old and the old could be lazily unmounted. Brauner thought that all of that should work with the existing mount-beneath feature. 

There was some discussion between Viro and Brauner about the propagation problem that was demonstrated. Brauner avoids that in his patches for mount beneath by simply returning an error if this mount-propagation explosion is going to happen. Viro did not seem to be opposed to that approach. 

Brauner struggled to describe some of the scenarios that could occur, not because he did not understand them, but because it is difficult to do so in words with limited examples from his computer screen. Viro cautioned that it would be extremely important to fully document the intended behavior, corner cases, and such, because reconstructing them from the code "will be unbearably hard". Brauner said he had a 1600-line file that describes all of the corner cases, just for his own reference; he agreed that comments in the code and documentation will be imperative. 

Brauner also poked Howells about his promise to provide documentation for the new mount API system calls. Brauner said that he has been a strong proponent of switching user-space programs to use the new API, has made the switch for a few projects, and that other projects (e.g. systemd) had switched as well; one of the main stumbling blocks is that he has to spend a lot of time explaining how the new system calls work. Viro apologized, though Brauner (and Howells) seemed to think the fault lay elsewhere. With luck, that gentle prod will spur work to finish up the documentation and get it merged.
