---
title: Supporting large block sizes
url: https://lwn.net/Articles/933437/
date: "June 5, 2023"
category: Block layer
author: "By Jake Edge June 5, 2023 LSFMM+BPF"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
June 5, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

At the [2023 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023>), Luis Chamberlain led a plenary session on kernel support for block sizes larger than 4KB. There are assumptions in the current kernel that the block size used by a block-layer device is less than or equal to the system's page size—both are usually 4KB today. But there have been efforts over the years to remove that restriction; that work may be heading toward fruition, in part because of the folio efforts of late, though there are still lots of areas that need attention. 

Originally, storage devices used 512-byte blocks, but over time that has grown to 4KB and beyond, Chamberlain said. Supporting block sizes greater than the page size has been desired for years; the first related patches were [posted 16 years ago](<https://lwn.net/Articles/232757/>) and the topic comes up at every LSFMM, he said. There is a [wiki page](<https://kernelnewbies.org/KernelProjects/large-block-size>) about the project as well. 

[ ![\[Luis Chamberlain\]](https://static.lwn.net/images/2023/lsfmb-chamberlain-sm.png) ](<https://lwn.net/Articles/933624/>)

XFS has supported 64KB blocks for quite some time, but only on systems that have 64KB pages; he believes some PowerPC-based systems were shipped with XFS filesystems using 64KB blocks. But the original goal of the long-ago patch set is the same as today: to support 64KB blocks (and other power-of-two sizes) on systems with 4KB pages. To that end, he has added an experimental option to his [kdevops](<https://github.com/mcgrof/kdevops>) tool that will create NVMe devices with larger block sizes. You can format the devices and boot a system, but if the devices are enabled, the system crashes. Trying to solve that problem is how he got involved in this work. 

He has a [Git tree](<https://git.kernel.org/pub/scm/linux/kernel/git/mcgrof/linux-next.git/log/?h=large-block-next>) for collecting patches of interest for the effort. Beyond just creating the large-block devices, he has also added [ways to test modified kernels in kdevops](<https://github.com/linux-kdevops/kdevops/blob/master/docs/lbs.md>). Currently development and testing is ongoing using XFS. 

Chamberlain is tracking the effort using "objectives and key results" (OKRs) in [a spreadsheet](<https://docs.google.com/spreadsheets/d/e/2PACX-1vS7sQfw90S00l2rfOKm83Jlg0px8KxMQE4HHp_DKRGbAGcAV-xu6LITHBEc4xzVh9wLH6WM2lR0cZS8/pubhtml#>); he is aware that some hate OKRs, but he is simply using them as a tracking tool for himself, though others are welcome to use them as well. In various parts of the talk, he displayed the OKR lists, starting with the list for converting to the iomap support layer. 

The previous session on [buffer-head removal](<https://lwn.net/Articles/931809/>), which was led by Hannes Reinecke, did not really address the block-device (bdev) cache, Chamberlain said. The bdev cache is important because some filesystems use it for metadata, and it uses buffer heads. As Reinecke said in that session, though, there may never come a time when buffer heads will be fully removed from the kernel. 

For filesystems that want to support large block sizes, the right path is to use iomap, Chamberlain said. It will take a while for iomap to be fully ready to support block sizes larger than the page size, but there is a path to get there, he believes. The block layer itself still requires some work in order to support the larger sizes, contrary to Matthew Wilcox's assertion that nothing more was needed, Chamberlain said. There is agreement now that only 0-order folios will be allowed for buffer heads, which removes one of the entries from his list. 

Ted Ts'o wanted to understand the business case for supporting these larger block sizes; he went back to the [email proposing the LSFMM topic](<https://lwn.net/ml/linux-fsdevel/ZAJqjM6qLrraFrrn@bombadil.infradead.org/>), which seemed to indicate the push is coming from the storage vendors. He asked: what are the use cases where a 32KB or 64KB block size makes sense? If he is to ask his company for time to work on this support, he needs to be able to justify it and he felt that part was left out of the discussion. 

James Bottomley pointed out that Ts'o had mentioned 16KB database blocks in his [session the previous day](<https://lwn.net/Articles/932900/>), but Ts'o said that the [atomic-write support](<https://lwn.net/Articles/933015/>) was a path to being able to write 16KB blocks without tearing (i.e. partial writes). Atomic writes can come in a reasonable time frame, and do not require the large-block support, which is a more sprawling effort. Reinecke said that there was a simple answer to the objection from Ts'o: if he could not justify working on large-block-size support, "then don't". 

Reinecke said that supporting these larger blocks is something of an experiment. There is a belief that it will lead to better performance, but the only way to find out is to try it. Ts'o said that it seems like a huge project, so he would need to be able to justify putting people to work on it. Josef Bacik said that the session was not aimed at making that justification, rather it is trying to see what the status and plans are for the project. He suggested that the group move back on track. 

There are a bunch of ancient filesystems in the kernel, Chamberlain said, that are not going to be updated; some, such as ReiserFS, are already slated for removal. There are others that might be removed, but supported via FUSE. He thinks that there are some old filesystems lacking `mkfs` tools, which makes them hard to test, thus hard to support. He thought it would be good to put together a plan for what to do with various old, likely unmaintained, filesystems. 

Supporting folios larger than order-0 (single page) in filesystems is needed, he said, but there are questions about what needs to be done for memory compaction. Wilcox said that the memory compaction code has not yet been converted from pages to folios, so that needs to happen. There is also a need to be able to migrate larger folios (not order-0) from one zone to another to try to ensure that higher-order allocations will not fail. Once the conversion to folios happens, he or someone else can dig more into the migration issue. 

Chamberlain said that he has been working with Dave Chinner on rebasing Chinner's older patch set that [added support for block sizes larger than the page size for XFS](<https://lwn.net/ml/linux-fsdevel/20181107063127.3902-1-david@fromorbit.com/>); the needed changes now just boil down to two patches. Testing is ongoing to ensure that those patches have not broken anything. Once that has been established, the next step would be to test XFS on a real device with a larger block size. 

Chamberlain wondered if other filesystems were interested in supporting larger block sizes. Bacik said that Btrfs already handles metadata in 16KB blocks, so he would like to make that work on larger-block devices. He would love to do it for data too, but work on that will not happen until after the iomap conversion for Btrfs data is done. 

Bottomley said that Chinner's original patch set was much larger than two patches, so he wondered whether that was due to the folio conversion that had already gone into XFS. If so, what looks like a huge amount of work for supporting large block sizes may turn out to be relatively straightforward. Chamberlain confirmed that; for filesystems that want to support larger block sizes, it is much easier once the folio conversion has been done. 

Handling metadata for those filesystems that are still using buffer heads may still be an issue, however. From afar, Darrick Wong noted that XFS has its own buffer cache internally, so its metadata handling can already use block sizes larger than the page size—"at least until memory fragmentation kills you". The part that does not work right now is that iomap is lacking a way to tell the memory-management subsystem that XFS needs multi-page folios that are at least of a certain size. 

After some discussion of memory fragmentation issues, Wong suggested that someone should simply set up a modified XFS using 8KB blocks (and 4KB pages) in order to run MySQL on a system without much memory. The idea would be to see if it falls over any faster than a regular XFS with 4KB blocks. Chamberlain said that the issue of how to test these changes is one that needs to addressed; a test plan with specifics about how to measure the impacts (good or bad) of the changes is needed.
