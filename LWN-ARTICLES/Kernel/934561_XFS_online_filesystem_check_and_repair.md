---
title: XFS online filesystem check and repair
url: https://lwn.net/Articles/934561/
date: "June 15, 2023"
category: "Filesystems-XFS"
author: "By Jake Edge June 15, 2023 LSFMM+BPF"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
June 15, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

Darrick Wong has been [doing work on XFS online repair](<https://lwn.net/Articles/754504/>) for a number of years and things are getting to the point where most of the filesystem-internal work has been completed and is under review. The work remaining mostly concerns the user-space side to set up a periodic scan and repair cycle, so he wanted to discuss what user space needs from this kind of feature in a filesystem session at the [2023 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023>) that he led remotely. The session may not have gone quite as he hoped, as it got somewhat derailed by topics that spilled over from the [earlier session on unprivileged image mounts](<https://lwn.net/Articles/934176/>). 

His current patch set for XFS online repair is "out for review on Dave Chinner's laptop right now", so it is time to start talking about the missing pieces. That means that he will be talking more about user space than he would normally; there is a user-space driver program that controls how often the online `fsck` mechanism runs. There is nothing yet for notifying user space of problems that were found by an online `fsck` pass, nor is there a daemon monitoring for notifications to do anything about them, such as to issue repair requests. There is no good infrastructure in the kernel for handling and dispatching such things, he said. 

[ ![\[Darrick Wong\]](https://static.lwn.net/images/2023/lsfmb-wong-sm.png) ](<https://lwn.net/Articles/934562/>)

He said that the earlier discussion in the unprivileged-mounts session on using `fsck` to decide that an image was sound enough to mount made him think that it was a good time to discuss these kinds of issues. 

As he noted, there is a command-line program, [`xfs_scrub`](<https://man7.org/linux/man-pages/man8/xfs_scrub.8.html>), which opens the block device and root directory, then starts issuing the right `ioctl()` commands, but the real use case is not for running a tool in that fashion. Instead, the idea is that it would do a background check and repair periodically from a systemd service; he is struggling a bit with setting that up, but has something working. It is not, however, much different from the age-old periodic cron job that reports its results to the system log and hopes an administrator is paying attention. 

He would like to create a notification system that would allow the system to respond dynamically to the events that get reported by the periodic scrubbing. He would also like there to be a way for programs to initiate scrubbing for various reasons, such as a container manager that notices relatively low activity so it kicks off scrubbing on the mounted filesystems. Maybe that could mesh with the unprivileged-mounting use case in some fashion as well, Wong said. 

So he wondered if any user-space developers had thoughts on how they might want to use this facility. He could continue developing "with my kernel colored glasses on", but he fears that may not produce the best results. There was an effort made to scare up Lennart Poettering, who might have some thoughts on the matter, but who had not made it back to the filesystem room after the coffee break. 

Josef Bacik said that he generally relied on people from Fedora and other distributions to give him feedback on features of this sort. The distribution developers often have different ideas on how these things will be used. So, for thoughts on policies that might be applied to the online scrubber, he recommended seeking out people from Linux distributions. 

Ted Ts'o replayed some of the earlier discussion around using (offline) `fsck` to check image files before mounting them. In order to be sure that image files are not modified by user space while the `fsck` was being done, Ts'o had said that they would need to be copied somewhere inaccessible to user space beforehand. One difference with the in-kernel `fsck` equivalent that XFS is planning to add might be that the copy/snapshot step would be unnecessary. A kernel-level `fsck` might not have that requirement, he suggested, but that does not really change whether using `fsck` in that manner is sufficient. 

By that point, Poettering had returned so Wong repeated some of what he had said earlier. He said that the work on the online scrubber had quite recently become more urgent because "a lot more distros than the zero I thought there were will actually let you mount XFS filesystems without privilege". There have also been recent efforts in XFS to flag strange problems ("weird-looking metadata or outright bad metadata") that it sees, but that is not connected with fsnotify events (as ext4 is) to notify user space of these kinds of corruption. XFS generally knows exactly what the problem was, which could be encoded in the notification somehow in the hopes that someone is listening and can take appropriate action. For some filesystems that might be to unmount and `fsck` the filesystem, while XFS could use the online repair facility. 

Poettering said that the current practice of having desktops mount removable media automatically is "stupid"; the approach that Chrome OS takes with only mounting certain specific filesystem types (e.g. VFAT), and only through a user-space driver, is much better and one that other desktops should adopt. The desktop use case is generally for USB sticks, and people do not normally put XFS on that kind of media, he said, so those should not be automatically mounted at all. 

For mounting filesystem images in containers, though, he thinks trust should come from dm-verity as described in his earlier talk. Ts'o had said that `fsck` might be sufficient for establishing that an ext4 image would not compromise the kernel, so Poettering wondered if Wong would say the same for XFS. There is a difficult answer to that, Wong said; "as soon as I say 'yes', everybody in the world will watch their fuzzer rigs in order to try to find all of the things that `fsck` doesn't catch". That said, he generally agrees with Ts'o that `fsck`, either online or offline, should be robust enough to catch any bad filesystems, but it is not an absolute guarantee since bugs happen. 

Poettering noted that the online checking for XFS was not usable for establishing trust, since the filesystem would need to be mounted first. Wong agreed, but wondered about images that had been signed by the distributor. Poettering and Christian Brauner said that signed images are fully trustable or, at least, that it is a user-space problem if they are not. Kent Overstreet said that `fsck` could not be used to establish trust in any case because a malicious device could change the data out from underneath the check. While that is true for, say, USB devices, the snapshot/copy requirement for a local image file that is getting mounted in a container removes that possibility, Ts'o said. 

Overstreet argued that requiring the copy was onerous and unenforceable for users. Instead, he thinks "the responsible thing for us to be doing as filesystem implementers is to start taking it a little bit more seriously to just hardening our code at run time". He said that XFS does a lot of read- and write-time verification of metadata along with fuzzing, as does Overstreet's [bcachefs](<https://bcachefs.org/>), so "we might not be in as bad a shape as we assume". 

Brauner wanted to clarify that the copy and `fsck` being discussed was not something that is under the user's control, but would be handled by a mount daemon. Overstreet was adamant that it would still be unacceptable to do the copy and "people are going to want to be able to mount images in the cloud untrusted very soon". 

Bacik said that the session was "getting off the rails" at that point. He said that Wong is interested in what kinds of notifications would be of interest to user space and how to handle the policy questions around those; Wong agreed with that. Poettering said that he is "not a storage guy" so he does not know what kinds of policies they might want, but he thinks that simply shutting down the affected services when a filesystem it relies on has errors is the safest approach. If systemd were to get a notification of that sort, it could easily be set up to shut down affected services. 

Ts'o said that those who are running these kinds of services should be consulted about how to handle the events. For example: what do the Kubernetes people actually want? They may want to shut down affected services, but give the services a short time frame to send a "goodbye cruel world" message or similar. The ext4 notifications that Wong mentioned were specifically added for the internal Google Kubernetes-like container manager Borg; the people maintaining those systems wanted to be able to shut down services in the face of filesystem corruption. 

Wong said things are a little different working for a large database vendor (Oracle); most of the use of XFS, beyond root filesystems, is for "really large data partitions where we would like to be able to perform at least simple repairs on the 100TB data partition to try to keep the VM running". At any given time, the workload running in the VM or container is probably not accessing the whole 100TB, so there is an opportunity to fix things before the application even notices. "We would at least like to try to grow new engines on the plane while it's flying in order to avoid having to do an emergency landing." Restoring 100TB (or even more) can take a long time, which is best avoided. 

Poettering wondered if a mount option that simply instructed XFS to run its online scrubber whenever it detected an anomaly might be a reasonable approach. "Why involve user space to trigger the online filesystem check?" User space is better for performing actions on other parts of the system, such as shutting down relevant services, so it does not really make sense for XFS to notify of a problem and have user space say "go fix yourself". Wong said that he was willing to write an XFS daemon that would receive notifications and schedule scrubbing if need be. 

He wrapped up by describing some of the fuzzing that is done for XFS, which uses the XFS debugger to "walk every single field of every metadata object in the entire filesystem and fuzz them". That is part of why the XFS QA test suite takes almost a week to run; it spends a lot of time fuzzing and checking to see that the repair tool notices the problems and can fix them, both online and offline. He thinks he added some fuzzing of ext4 metadata blocks to fstests along the way, but not to the same level of precision of the XFS fuzz testing.
