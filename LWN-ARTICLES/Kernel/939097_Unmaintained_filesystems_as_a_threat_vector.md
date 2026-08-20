---
title: Unmaintained filesystems as a threat vector
url: https://lwn.net/Articles/939097/
date: "July 28, 2023"
category: "Filesystems-Security; Security"
author: "By Jonathan Corbet July 28, 2023"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
July 28, 2023

One of the longstanding strengths of Linux, and a key to its early success, is its ability to interoperate with other systems. That interoperability includes filesystems; Linux supports a wide range of filesystem types, allowing it to mount filesystems created by many other operating systems. Some of those filesystem implementations, though, are better maintained than others; developers at both the kernel and distribution levels are currently considering, again, how to minimize the security risks presented by the others. 

#### HFS (and HFS+) in the kernel

Back in January, the syzbot fuzzing system [reported](<https://lwn.net/ml/linux-kernel/000000000000dbce4e05f170f289@google.com/>) a crash with the HFS filesystem. For those who are not familiar with HFS, it is the native filesystem used, once upon a time, by Apple Macintosh computers. Its [kernel configuration help text](<https://elixir.bootlin.com/linux/v6.4.5/source/fs/hfs/Kconfig>) promises that users ""will be able to mount Macintosh-formatted floppy disks and hard drive partitions with full read-write access"". It seems that, in 2023, there is little demand for this capability, so the number of users of this filesystem is relatively low. 

The amount of maintenance it receives is also low; it was [marked as orphaned](<https://git.kernel.org/linus/6cf515e113fc>) in 2011, at which point it had already seen some years of neglect. So it is not all that surprising that the syzbot-reported problem was not fixed or, even, given much attention. At the end of the brief discussion in January, Viacheslav Dubeyko, who occasionally looks in on HFS (and the somewhat more modern HFS+ filesystem as well), [said](<https://lwn.net/ml/linux-kernel/50D6A66B-D994-48F4-9EBA-360E57A37BBE@dubeyko.com/>) that there was nothing to be done in the case where a filesystem has been deliberately corrupted. 

On July 20, Dmitry Vyukov (who runs syzbot) [restarted the discussion](<https://lwn.net/ml/linux-kernel/CACT4Y+aJb4u+KPAF7629YDb2tB2geZrQm5sFR3M+r2P1rgicwQ@mail.gmail.com/>) by pointing out that the consequences of a bug in HFS can extend beyond the small community of users of that filesystem: ""Most popular distros will happily auto-mount HFS/HFS+ from anything inserted into USB (e.g. what one may think is a charger). This creates interesting security consequences for most Linux users"". There is an important point in that message that is worth repeating: users may not be aware that the device they are plugging into their computer contains a filesystem at all. One often sees warnings about plugging random USB sticks into a computer, but any device — or even a charging cable — can present a block device with a filesystem on it. If the computer mounts that filesystem automatically, ""interesting security consequences"" may indeed follow. 

The new round of discussion still has not resulted in the problem being fixed. Instead, some developers called for the removal of the HFS and HFS+ filesystems entirely. Matthew Wilcox [said](<https://lwn.net/ml/linux-kernel/ZLlvII%2FjMPTT32ef@casper.infradead.org/>): ""They're orphaned in MAINTAINERS and if distros are going to do such a damnfool thing, then we must stop them"". Dave Chinner [argued](<https://lwn.net/ml/linux-kernel/ZLmzSEV6Wk+oRVoL@dread.disaster.area/>) that the kernel community needs to be more aggressive about removing unmaintained filesystems in general: 

> We need to much more proactive about dropping support for unmaintained filesystems that nobody is ever fixing despite the constant stream of corruption- and deadlock- related bugs reported against them. 

Linus Torvalds, though, [was unimpressed](<https://lwn.net/ml/linux-kernel/CAHk-=wg7DSNsHY6tWc=WLeqDBYtXges_12fFk1c+-No+fZ0xYQ@mail.gmail.com/>), saying that, instead, distributors should just fix the behavior of their systems. The lack of a maintainer, he added, is not a reason to remove a filesystem that people are using; ""we have not suddenly started saying 'users don't matter'"". That brought the discussion to an end, once again, with no fix for the reported bug in sight. 

#### Distribution changes

As the conversation was reaching an end on the linux-kernel list, it [picked up](<https://lwn.net/ml/debian-devel/ZLl1PLU938klkucC@bongo.bofh.it/>) on debian-devel. There, Marco d'Itri asked the kernel developers to simply blacklist HFS and HFS+ from being used for automounting filesystems. Matthew Garrett, though, [pointed out](<https://lwn.net/ml/debian-devel/20230721082012.GA19433@srcf.ucam.org/>) that the kernel, which cannot completely block automounting without disabling the filesystem type entirely, was probably the wrong place to solve the problem. Instead, he [suggested](<https://lwn.net/ml/debian-devel/20230721173536.GA26591@srcf.ucam.org/>), a udev rule could be used to prevent those filesystems from being automounted, while keeping the capability available for users who manually mount HFS or HFS+ filesystems. 

Shortly thereafter, Garrett [raised the issue](<https://lwn.net/ml/fedora-devel/ZLtwvqu%2FCBaN1iiX@srcf.ucam.org/>) on the Fedora development list as well, suggesting the addition of a udev rule once again. There, some participants saw that rule as perhaps improving the situation, but others, including [Zbigniew Jędrzejewski-Szmek](<https://lwn.net/ml/fedora-devel/ZLu+Cm13gm%2F9whyZ@kawka3.in.waw.pl/>) and [Michael Catanzaro](<https://lwn.net/ml/fedora-devel/RZ77YR.54R774W5MQTA@redhat.com/>), pointed out that, if a user wants to see the files contained within a a filesystem image, they will do what is needed to mount it, even if that mounting does not happen automatically. Solomon Peachy [suggested](<https://lwn.net/ml/fedora-devel/ZL1Q3eTS2c9A+Ee2@shaftnet.org/>) that adopting this policy would only result in an addition to the various "things to fix after installing Fedora" lists telling users how to turn automounting back on. 

Nobody mentioned the possibility that the user was not expecting a given device to have a filesystem at all. Forcing such a filesystem to be mounted manually would presumably address that problem since, presumably, most users would not go to the trouble of mounting a filesystem that they did not expect to be there in the first place. But, as Demi Marie Obenour [pointed out](<https://lwn.net/ml/fedora-devel/fb671fde-b24a-0ba4-f507-5dd5ce00b137@gmail.com/>), a malicious filesystem image could be employed willingly by a user to take control of a locked-down system: 

> Unfortunately, this original threat model is out of date. kernel_lockdown(7) explicitly aims to prevent root from compromising the kernel, which means that malformed filesystem images are now in scope, for all filesystems. If a filesystem driver is not secure against malicious filesystem images, then using it should fail if the kernel is locked down, just like loading an unsigned kernel module does. 

In that case, it seems, disabling automounting would not be a sufficient fix; the vulnerable filesystem type would need to be disabled entirely. 

There is an aspect of the problem that has not received as much attention as it might warrant, though Eric Sandeen did [touch on it](<https://lwn.net/ml/fedora-devel/e16e3a56-cd51-3bd5-05a9-10a1f0b74652@redhat.com/>): the number of filesystem implementations in Linux that are robust in the face of a _maliciously_ corrupted image is quite close to zero. Many filesystems can deal with corruption resulting from media errors and the like; checksums attached to data and metadata will catch such problems. Malicious corruption, instead, will have correct checksums, entirely bypassing that line of defense. Filesystem developers who have thought about this problem are mostly unanimous in saying that it cannot readily be solved — the space for possible attacks is simply too large. 

So, while unmaintained filesystems like HFS may provide a sort of low-hanging fruit for attackers, they are not the sole cause of the problem. Intensively maintained filesystems, including ext4, Btrfs, and XFS, are also susceptible to malicious filesystem images. So even removing support entirely for the older, unmaintained filesystem types would not solve the problem. 

In the Debian discussion, Garrett suggested that risky filesystems could be mounted as FUSE filesystems in user space, thus making it much easier to contain any ill effects — ""but even though this has been talked about a bunch I haven't seen anyone try to implement it"". On the Fedora side, Richard W. M. Jones [suggested](<https://lwn.net/ml/fedora-devel/20230722105247.GC7781@redhat.com/>) that [libguestfs](<https://libguestfs.org/>), which mounts filesystems within a virtual machine, could be used. Once again, that would contain the results of any sort of exploitation attempt. 

If the objective is truly to make it safe for users to mount untrusted filesystems, some sort of isolation will almost certainly prove to be necessary. Making most filesystem implementations robust against malicious filesystem images just does not seem to be an attainable goal in the near future — even if resources were being put toward that goal, which is not happening to any great extent. It is not a simple solution, and the result will have a performance cost, but security often imposes such costs.
