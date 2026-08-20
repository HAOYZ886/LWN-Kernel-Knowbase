---
title: Flash storage topics
url: https://lwn.net/Articles/756627/
date: "June 6, 2018"
category: "Filesystems-Flash"
author: "By Jake Edge June 6, 2018 LSFMM"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
June 6, 2018

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2018/>)

At the 2018 Linux Storage, Filesystem, and Memory-Management Summit (LSFMM), Jaegeuk Kim described some current issues for flash storage, especially with regard to Android. Kim is the [F2FS](<https://en.wikipedia.org/wiki/F2FS>) developer and maintainer, and the filesystem-track session was ostensibly about that filesystem. In the end, though, the talk did not focus on F2FS and instead ranged over a number of problem areas for Android flash storage. 

He started by noting that [Universal Flash Storage](<https://en.wikipedia.org/wiki/Universal_Flash_Storage>) (UFS) devices have high read/write speeds, but can also have high latency for some operations. For example, ext4 will issue a discard command but a UFS device might take ten seconds to process it. That leads the user to think that Android is broken, he said. 

[ ![\[Jaegeuk Kim\]](https://static.lwn.net/images/2018/lsf-kim-sm.jpg) ](<https://lwn.net/Articles/756687/>)

UFS devices have a "huge garbage-collection overhead". When garbage collection is needed, the performance of even sequential writes drops way down. That needs to be avoided, so UFS must be periodically given some time to do its garbage collection. But power is a more important consideration, so hibernating the device is prioritized, which does not leave much time for the device to do its garbage collection. 

Amir Goldstein suggested doing garbage collection when the device is charging; he thought that should provide a reasonable solution. Kim said that Android currently declares a ten-minute idle time at 2am that is used to defragment the filesystem. It could perhaps also be used for garbage collection. 

The solution to the discard performance problem should be fairly straightforward, he said. A kernel thread (kthread) could be added to issue discards asynchronously during idle time. Candidate blocks could be added to a list that would be processed by the kthread. There is a race condition if the block gets reallocated, however. 

Different UFS devices have different latencies for their cache-flush commands. Some vendors' devices have low latency but others have ten-second latencies for a single cache-flush command. Given that, it makes sense to batch cache-flush commands. 

Filesystem encryption is mandatory for Android. It is present in ext4 and has also been added to F2FS. There is some hardware encryption code from Qualcomm that cannot be pushed upstream, however. Ted Ts'o said that it is "horrible code" that only works for ext4 ecryptfs or F2FS; no one has had time to clean it up for the mainline. 

Kim would like to see the garbage collection on the device side get optimized. He would like to add a customized interface that can be called when it is time to do garbage collection. If the system can detect idle time, it can then initiate the garbage-collection process. 

SQLite performance is another problem area. SQLite uses `fsync()` to ensure its data has gotten to storage. By default it uses a journal, so writes to the database end up requiring two writes and two `fsync()` calls (first for the journal and then to the final location). Two `fsync()` operations can be expensive and are not needed for F2FS because it is a copy-on-write filesystem. A feature has been added to SQLite to avoid one write and one `fsync()` by using F2FS atomic writes. 

In order to reduce the latency of `fsync()` calls, he is looking at write barriers. He researched them and found that they had been [removed long ago](<https://lwn.net/Articles/400541/>). Kent Overstreet said they were removed due to unclear semantics, especially for stacked filesystems. In that case, the stack would have to provide order guarantees for the BIOs all the way down the stack, which would be difficult to do and would defeat the purpose of some of the layers. Beyond that, it is impossible to test to make sure that has been done correctly. 

But Kim said that the Android case would not involve device-mapper or other stacking, he is just trying to avoid the cache-flush command. Jan Kara suggested a new storage command, like "issue barrier", that would cause any I/O issued before the barrier to complete before any new I/O.
