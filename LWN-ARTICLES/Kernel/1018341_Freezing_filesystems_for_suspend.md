---
title: Freezing filesystems for suspend
url: https://lwn.net/Articles/1018341/
date: "April 24, 2025"
category: Filesystems
author: "By Jake Edge April 24, 2025 LSFMM+BPF"
---

By **Jake Edge**  
April 24, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

Sometimes worms have a tendency to multiply once their can is opened. James Bottomley recently encountered that situation; he led a session in the filesystem track at the 2025 Linux Storage, Filesystem, Memory Management, and BPF Summit (LSFMM+BPF) to discuss filesystem behavior with respect to suspending and resuming the system. As he noted in his [topic proposal](<https://lwn.net/ml/all/0a76e074ef262ca857c61175dd3d0dc06b67ec42.camel@HansenPartnership.com/>), he came at the problem because he needed a way to resynchronize the contents of [efivarfs](<https://www.kernel.org/doc/html/latest/filesystems/efivarfs.html>) after a system resume and thought there should be an API available to use. But, as the resulting thread shows, the filesystem freeze and thaw code had never been used by the system-wide suspend and resume code. Due to a scheduling mixup, though, several of us missed Bottomley's session, including Luis Chamberlain who has been working on hooking those two pieces up; what follows is largely from a second session that Chamberlain led, with some background information from the topic-proposal discussion and an email exchange with Bottomley. 

#### Background

The underlying problem that Bottomley is trying to solve is that efivarfs may not reflect the correct state of the EFI variables on the system after a resume operation. That's because some other OS could have been booted on the system after the suspend and changed some of the variables. So he was looking for a hook where he could add a resync operation that would run when the system is resumed. He had a solution, though it was kind of ugly, but he thought other filesystems might benefit from having some kind of API in order to give better guarantees about filesystem consistency across a suspend-resume cycle. As his proposal notes: 

> Hibernate is a particularly risky operation and resume may not work leading to a full reboot and filesystem inconsistencies. In many ways, a failed resume is exactly like a system crash, for which filesystems already make specific guarantees. However, it is a crash for which they could, if they had power management hooks, be forewarned and possibly make the filesystem cleaner for eventual full restore. Things like guaranteeing that uncommitted data would be preserved even if a resume failed, which isn't something we guarantee across a crash today. 

The API for freezing and thawing superblocks was believed to be the proper hooks to use, but that API is not called by suspend and resume, though many were under the impression that it was. In the proposal thread, Bottomley noted that there were [several attempts](<https://lwn.net/ml/all/576418420308d2511a4c155cc57cf0b1420c273b.camel@HansenPartnership.com/>) to freeze filesystems when suspending, but they ran aground on deadlock problems of various sorts. Jan Kara [described a problem for FUSE filesystems](<https://lwn.net/ml/all/vnb6flqo3hhijz4kb3yio5rxzaugvaxharocvtf4j4s5o5xynm@nbccfx5xqvnk/>); user space needs to still be running or writes will block, so there are some ordering issues that need to be worked out. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

As part of the first session, it was agreed that it would make sense to try freezing the filesystems in reverse superblock order, so that each level closer to the storage would still be active so that data could be flushed. Any user-space processes that are accessing the filesystems after they have been frozen will effectively suspend themselves and user space can be fully suspended after all of the filesystems are frozen. It was generally believed that would all work, but it would obviously need to be tested, which Bottomley [started working on](<https://lwn.net/ml/all/25f2c7dd471077d816ad29e461c6d762921927e1.camel@HansenPartnership.com/>) during the summit. 

#### Second session

Chamberlain has been working on freezing filesystems for a few years now. He said that he was working on the patches, which he had [last posted back in 2023](<https://lwn.net/ml/all/20230508011717.4034511-1-mcgrof@kernel.org/>), on the plane to the summit, but did not finish. His posting was made, in part, to prepare attendees for his [session at LSFMM+BPF 2023](<https://lwn.net/Articles/935602/>). At the time of that posting, there were some locking problems that he had [worked around](<https://lwn.net/ml/all/20230508011717.4034511-2-mcgrof@kernel.org/>), which he raised at the start of this year's session. Christian Brauner said that those problems had already been fixed; there were some locking inversions between the VFS and block layers for freezing, but it has all been untangled ""and should be safe from that perspective, as far as I can tell"". 

The rest of Chamberlain's patches are all Coccinelle-driven and fairly straightforward, he said. After applying those, testing with XFS could commence, followed by adding them to ext4 and other filesystems, one by one, with, of course, more testing. There was some discussion of the ordering of freezing the filesystems, including various possible deadlock problems, but most of the problematic cases had already been addressed or were on their way toward removal from the kernel. 

Automated testing of the change will be somewhat difficult, Chamberlain said, because there is no real framework for that. Fstests does not test the resume operation, for one thing. Currently the best way is to use a laptop for the testing, he said, so that there is access to a real system suspend and resume. Ted Ts'o suggested testing a loopback filesystem using a file on one filesystem that gets mounted on another as a good way to try to ensure that the ordering of the freeze operations is being handled correctly. 

Lennart Poettering said that systemd has given up calling [`sync()`](<https://www.man7.org/linux/man-pages/man2/sync.2.html>) before suspending the system due to problems with complicated mount topologies, often involving NFS, that take too much time to complete the operation. The project would have investigated freezing filesystems, but that is not something that user space can do because of the deadlock possibilities. He would be happy to see the kernel take care of all of that. 

Chamberlain returned to the plan to get rid of the kthread freezer (which was the topic of his session in 2023). The first step is to remove its use by filesystems, which is what his Coccinelle scripts do, but then there are other users of the kthread-freezer API that need to be addressed. The API was added for filesystems to use, but other parts of the kernel have started using it as well, so the ""next challenge"" will be getting rid of it throughout the kernel, he said. 

There was a lot of confusion in the audience about what the kthread freezer actually is. Bottomley suggested that it was the [`freeze_task()`](<https://elixir.bootlin.com/linux/v6.14.3/source/kernel/freezer.c#L152>) call, but could not find any uses of that by filesystems. Maybe things have changed, Chamberlain said, but it is something that needs more investigation once filesystems are properly frozen during suspend. 

Over the remote link, Kara raised a potential problem with the proposed mechanism: loopback block devices that are not mounted but just have a file as their storage. Those devices may have dirty data but when that data is flushed, the underlying filesystem will be frozen, because block devices are frozen after filesystems, thus there will be a deadlock. That use case was deemed to be sufficiently rare that it could perhaps be ignored, at least for now. 

Since the session, there have been several patch postings, including [one from Chamberlain](<https://lwn.net/ml/all/20250326112220.1988619-1-mcgrof%40kernel.org/>) and [another from Brauner](<https://lwn.net/ml/all/20250401-work-freeze-v1-0-d000611d4ab0@kernel.org/>) that incorporates Chamberlain's patches.
