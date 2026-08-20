---
title: Sunsetting buffer heads
url: https://lwn.net/Articles/931809/
date: "May 18, 2023"
category: "Block layer-Buffer heads"
author: "By Jonathan Corbet May 18, 2023 LSFMM+BPF"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
May 18, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

The buffer head is a kernel data structure that dates back to the first Linux release; for much of the time since then, kernel developers have been [hoping to get rid of it](<https://lwn.net/Articles/930173/>). Hannes Reinecke started a plenary session at the 2023 [Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023>) by saying that everybody agrees that buffer heads are a bad idea, but there is less agreement on how to take them out of the kernel. The core functionality they provide — facilitating sector-size I/O operations to a block device underlying a filesystem — must be provided somehow. 

The key question, he said, was whether the existing buffer-head API should be reimplemented using folios or, instead, the best approach would be to just replace buffer heads with folios directly. One problem with the latter approach is that folios only support page-size I/O; pages are usually larger than sectors, so page-size I/O operations will necessarily transfer multiple sectors. It is not clear to him that this matters much, though. Filesystems work hard to pack data on disk, and chances are good that the adjacent sectors will also be accessed soon. With current hardware, he said, I/O size is no longer as important for performance, so filesystem developers should not hesitate to use page-sized I/O operations. 

[![\[Hannes Reinecke\]](https://static.lwn.net/images/conf/2023/lsfmm/HannesReinecke-sm.png)](<https://lwn.net/Articles/931915/>) An advantage that would come from using folios is that they can use the page cache directly. The buffer cache duplicates a fair amount of page-cache functionality; that code would be good to drop. Changing filesystems to _read_ page-sized chunks is relatively easy, he said, but the write side is less so. Filesystems often assume that a write will only affect one sector, but that is not the case with folios. There is [a patch series](<https://lwn.net/ml/linux-fsdevel/cover.1683208091.git.ritesh.list@gmail.com/>) from Ritesh Harjani adding support for sub-page dirty tracking that should help with the conversion process. 

Luis Chamberlain said that one possible sticking point is the block-device cache, which uses buffer heads for tasks like partition scanning. It's not clear that the benefits of converting this code are worth the risk. Reinecke answered that a full conversion away from buffer heads may never be necessary. Chamberlain said that a number of filesystems use buffer heads for metadata I/O and asked whether they wanted to move away from that; the answer appeared to be "yes". Jan Kara said that moving away from buffer heads was important, but that there is still a need for some sort of intermediate layer; Reinecke agreed that a drop-in replacement for buffer heads is important. 

Kara pointed out that filesystems use the buffer-head layer to maintain the association between sectors and the inode of the file containing them. It is, he said, "one of the darker corners" of the buffer-head code. Reinecke questioned whether that functionality was needed, since the dirtying of data happens at the folio level in any case. 

Ted Ts'o said that the buffer-head code hides a lot of functionality, and that each filesystem uses a different subset of that functionality. Sub-page tracking only replaces one piece of that puzzle. The ext4 filesystem still supports 1KB block sizes, so sub-page tracking can't be ignored; it needs that piece. Things get more complicated when small block sizes have to be supported on CPUs with 64KB base page sizes. 

Some filesystems also use the buffer cache to associate dirty buffers with inodes, Ts'o continued. Filesystem developers are slowly converting over to the iomap support layer, which makes that problem go away, but finishing the conversion is "a big lift". Some other buffer-cache functionality is only used by the journaled block device (JBD2) layer, and might be best just moved into that code. 

Yet another problem is that ext4 has to be able to support utilities that open the underlying block device while a filesystem is mounted; keeping everything coherent in that environment is a challenge, but the buffer cache makes it happen for free. He concluded by saying that there is value in replacing the ancient buffer-head code, even if it still provides functionality needed by filesystems, but it will be an incremental task. It will not be possible to just switch to folios for metadata; some sort of intermediate layer will still be needed. 

Josef Bacik agreed that "we all hate buffer heads", but said that every filesystem manages metadata in its own special way. Btrfs uses extent buffers that sit on top of `struct page` and will be converted to folios soon. XFS has its own structure for metadata. And so on. Just dropping buffer heads will not be the big win that everybody seems to think it will be, he said. A common replacement layer for filesystems is not the solution, since every filesystem is different. It would take some coercion to get him to switch Btrfs to a common layer. Reinecke answered that he is trying to outline a path by which filesystems can be converted away from buffer heads and is not trying to force anybody. 

Ts'o said that the buffer-head work won't matter for filesystems that do not use buffer heads now, but there are still quite a few that do use them, and the buffer-head layer provides an important support system. Bacik said that any conversion work should just be done within the buffer-head layer so that filesystems wouldn't notice the difference. Matthew Wilcox asked whether there was a desire for large folio support within the buffer head layer; that would be difficult to add transparently. Ts'o advised keeping things simple and stick to single-page folios for a replacement buffer-head layer. Filesystems that want multi-page support can switch to iomap. 

Bacik brought the session to a close by stating the apparent conclusion from the session: the buffer-head layer will be converted to use folios internally while minimizing changes visible to the filesystems using it. Only single-page folios will be used within this new buffer-head layer. Any other desires, he said, can be addressed later after this problem has been solved.
