---
title: Defending mounted filesystems from the root user
url: https://lwn.net/Articles/941764/
date: "August 21, 2023"
category: "Filesystems-Security; Security"
author: "By Jonathan Corbet August 21, 2023"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
August 21, 2023

Making a filesystem implementation robust in the face of maliciously created filesystem images is a challenging task even when the implementation is actively maintained, which [many in the kernel are not](<https://lwn.net/Articles/939097/>). There is a way to make that task even harder, though: modify that filesystem image behind the implementation's back while it is mounted. A recent discussion on the linux-fsdevel list reveals an ongoing disagreement over whether (and how) this threat should be addressed. 

Gabriel Krisman Bertazi recently posted [a patch series](<https://lwn.net/ml/linux-fsdevel/20230812004146.30980-1-krisman@suse.de/>) adding support for negative dentries on case-insensitive ext4 and F2FS filesystems. Negative dentries cache the results of lookups on files that do not exist, accelerating subsequent lookups. Since this kind of operation happens frequently (consider, for example, iterating through a `PATH` environment variable to find an executable), this is an important optimization. Currently, though, negative dentries do not work with case-insensitive filesystems; this patch series rectifies that problem. 

In the review discussion for this series, Eric Biggers [asked](<https://lwn.net/ml/linux-fsdevel/20230812015915.GA971@sol.localdomain/>) about a specific check for the case where an inode shows up with the case-insensitive flag set, even though the filesystem has not been mounted for case-insensitive operation. This check was [added by Ted Ts'o](<https://git.kernel.org/linus/6456ca6520ab>) in 2019 to fix a crash experienced while fuzzing the filesystem. Biggers wondered why the test was placed at the inode's point of use rather than when that inode is first read from the disk. 

Ts'o [answered](<https://lwn.net/ml/linux-fsdevel/20230812230647.GB2247938@mit.edu/>) that the inode can change _after_ it has been read into memory, in certain conditions: 

> It's not enough to check it in ext4_iget, since the casefold flag can get set *after* the inode has been fetched, but before you try to use it. This can happen because syzbot has opened the block device for writing, and edits the superblock while it is mounted. 

One might think that the case of writing to a mounted filesystem behind the implementation's back would be one of those "don't do that" situations. It is not an action that is going to lead to a satisfying conclusion. There is, however, disagreement over what should be done about this case; Ts'o continued: 

> One could say that this is an insane threat model, but the syzbot team thinks that this can be used to break out of a kernel lockdown after a UEFI secure boot. Which is fine, except I don't think I've been able to get any company (including Google) to pay for headcount to fix problems like this, and the unremitting stream of these sorts of syzbot reports have already caused one major file system developer to burn out and step down. 

Biggers [replied](<https://lwn.net/ml/linux-fsdevel/20230813043022.GA3545@sol.localdomain/>) that fixing problems caused this way is the wrong approach: 

> Well, one thing that the kernel community can do to make things better is identify when a large number of bug reports are caused by a single issue ("userspace can write to mounted block devices"), and do something about that underlying issue instead of trying to "fix" large numbers of individual "bugs". 

He pointed out that Jan Kara has posted [a patch set](<https://lwn.net/ml/linux-fsdevel/20230704122727.17096-1-jack@suse.cz/>) that addresses that issue by adding a configuration option to prohibit writing to block devices that are currently mounted. Applying this series — and configuring the kernel appropriately — would simply close off that entire avenue of attack and, Biggers said, make a large number of syzbot-reported bugs go away. 

There is a minor problem or two with this approach, though. One is that, as Kara describes in the cover letter, enabling this option breaks a number of things in both kernel and user space, including Btrfs mounting, loopback mounts, and filesystem resizing. Fixing these problems is seemingly not overly difficult, but one cannot just enable this option in the kernel until they _have_ been fixed, and those fixes have found their way onto deployed systems. That is a process that will take years. 

Even then, this series will prevent writing to a mounted _partition_ , but not to the device as a whole. If `/dev/sda1` is mounted it cannot be written to, but `/dev/sda` (which covers the whole device, including the `sda1` partition) is still fair game. And even if that were fixed, as Ts'o [pointed out](<https://lwn.net/ml/linux-fsdevel/20230704205529.GH1178919@mit.edu/>), there are other possible attacks, such as opening the SCSI-generic device and sending commands directly to the storage device. There is, it seems, always another way for a sufficiently privileged account to create mayhem. 

Yet another problem is that, [according to Ts'o](<https://lwn.net/ml/linux-fsdevel/20230814113852.GD2247938@mit.edu/>), the syzbot developers [are unwilling](<https://lwn.net/ml/linux-fsdevel/CACT4Y+aEScXmq2F1-vqAfr-b2w-xyOohN+FZxorW1YuRvKDLNQ@mail.gmail.com/>) to turn on this configuration option unless disabling it would be hidden behind a new `CONFIG_INSECURE` option (to indicate that doing so would make the system insecure). Ts'o objected to that positioning ""because that's presuming a threat model that we have not all agreed is valid"". 

So, even if Kara's series is applied to the kernel, it is a partial (albeit worthwhile) fix that cannot be enabled in deployed systems for years, and which will not be enabled by the people running the fuzzers. Filesystem developers will be limited to occasionally fixing symptoms of the problem as they appear while dealing with floods of fuzzing reports and questioning the basis on which these reports are made. It seems fair to say that this is not a great situation for anybody involved. 

The real problem, arguably, is that there is no consensus within the community regarding the threats that the kernel should try to address. A threat model that includes defending the system against its own root user will require a huge hardening effort that many developers feel is both impossible and pointless and which, in any case, does not have the funding it would need to have a chance at succeeding. The subset of the community that is pushing for this threat model thus finds itself in conflict with the rest. Resolving that disagreement may turn out to be the hardest problem of all.
