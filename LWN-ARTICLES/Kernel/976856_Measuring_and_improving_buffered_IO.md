---
title: Measuring and improving buffered I/O
url: https://lwn.net/Articles/976856/
date: "June 5, 2024"
category: Buffered IO
author: "By Jake Edge June 5, 2024 LSFMM+BPF"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
June 5, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

There are two types of file I/O on Linux, buffered I/O, which goes through the page cache, and direct I/O, which goes directly to the storage device. The performance of buffered I/O was reported to be a lot worse than direct I/O, especially for one specific test, in Luis Chamberlain's [topic proposal](<https://lwn.net/ml/linux-fsdevel/Zdkxfspq3urnrM6I@bombadil.infradead.org/>) for a session at the 2024 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>). The proposal resulted in a lengthy mailing-list discussion, which also came up in [Paul McKenney's RCU session](<https://lwn.net/Articles/974487/>) the next day; Chamberlain led a combined storage and filesystem session to discuss those results with an eye toward improving buffered I/O performance. 

#### Testing goals

He began by outlining his goals with the testing, which were to measure the limits of the page cache and to find ways that page-cache performance could be improved. In order to improve the performance, it needs to be measured; in particular, there needs to be a way to avoid introducing performance regressions as part of the work. He has done a lot of testing of the page cache, but there is a need to try to distinguish between normal and pathological use cases. 

Based on the tests that he has done, and a suggestion for a "normal" test case from Chris Mason, he wondered if it seemed reasonable to try to achieve throughput parity between buffered and direct I/O on a six-drive, RAID 0 configuration. Dave Chinner said "absolutely not"; he does not think it is possible to get parity between the two types of I/O in that configuration. Chamberlain suggested that the summit would be a good place to work out what the right tools, tests, and configurations would be to try to measure and improve page-cache performance (thus, buffered I/O performance). 

The pathological test case that he presented in the topic proposal is the one that got the most attention. On a well-resourced system, he reported 86GB/s writes for direct I/O and only 7GB/s writes using buffered I/O. That is a huge difference; he wondered whether it was acceptable or is something that should be investigated and, perhaps, fixed. 

Chamberlain said that there were some other outcomes from the thread. Matthew Wilcox [reported](<https://lwn.net/ml/linux-fsdevel/Zds8T9O4AYAmdS9d@casper.infradead.org/>) a problem with 64-byte random reads, which resulted in a [patch from Linus Torvalds](<https://lwn.net/ml/all/CAHk-=wjOogaW0yLoUqQ0WfQ=etPA4cOFLy56VYCnHVU_DOMLrg@mail.gmail.com>); Kent Overstreet [did some preliminary testing](<https://lwn.net/ml/all/yfzkhghkh36ww5nzmkdrdpcjy6w5v6us55ccmnh2phjla25mmz@xomuheona22l>) and found that it provided a 25% performance improvement. Torvalds was [not interested in pushing the patch](<https://lwn.net/ml/all/CAHk-=wi+Vcn0jhXqiwfK8eMXUOUqyU-uc+d-ACpZRjBs4SxiQA@mail.gmail.com>) any further, but Chamberlain said he is testing it some more to see that it does not crash. 

He described a few other things that came up in the thread, some of which were addressed with patches. But the results from the pathological case seem to be unexpected; what should be done about that? 

These kinds of discussions are always about tradeoffs, Ted Ts'o said; you trade some amount of safety, which you may not care about depending on the workload, for improvement on a microbenchmark. There may or may not be user-space applications that actually care about the operations measured in the microbenchmark, as well. For example, he does not know of any real-world application that needs to be able to do 64-byte random reads. 

It takes work to determine if these things make a difference to real-world applications, he continued, and more work to ensure that whatever changes are being considered do not break other applications and make things worse. There is a philosophical question that needs to be answered about whether it even makes sense to spend the time to investigate any given problem. 

He contrasted the problems being discussed with the [torn-write problem](<https://lwn.net/Articles/974578/>), which has a clear and obvious benefit for database performance if it gets solved; whether that is also true about providing high-performance for 64-byte I/O is unclear to him. In the absence of a customer, "is it worth it?" But Wilcox said that the 64-byte-read problem came from a real (and large) Linux-using customer. 

Ts'o's point is valid, Chamberlain said, but his goal in the session was not to come up with areas to address; he wanted to raise some of the questions that arose from his testing. 

#### Not unexpected

Chinner said that the numbers from the pathological test were not unexpected. Part of the problem is that buffered I/O has a single writeback thread per filesystem, so that I/O cannot go any faster than that. The writeback thread is CPU-bound; "it is not that the page cache is slow, it is cleaning the page cache that's slow", he said. There are hacks to get around that limitation somewhat, but what needs to be looked at is some way to parallelize cleaning the page cache. That could be multiple writeback threads, using writethrough instead, or some other mechanism; the architecture of the page cache is not scalable for the cleanup part. Writethrough means that writes go into the page cache, but are also written to storage immediately. 

Chamberlain wondered if there was general agreement with Chinner in the room. Wilcox said that he did not disagree, but that how the scaling is to be done is an interesting question. For example, a single huge file that needs writeback to be done in multiple places will be harder to scale than dealing with multiple small or medium-size files that need writeback. 

Chinner said that much of the CPU time in writeback is spent scanning the page cache looking for pages that need to be written, which does not really change based on the size of the files. There are some filesystem-specific considerations as well, but a pure-overwrite workload will have higher writeback rates because the amount of scanning needed is less; at that point it runs into contention for the LRU lock. Adding more threads into that mix will not help at all and may make things worse. 

One of the workloads that Chinner runs frequently simulates an untar operation with lots of files, each of which gets created, 4KB is written to it, and it gets closed. XFS gets stuck at around 50K files/s (roughly 200MB/s) on a device that can normally handle 7-8GB/s; the limitation is the single writeback thread. The rate goes way up (to 600K files/s or 2.4GB/s) if he does a flush when the file is closed, which simulates a writethrough mechanism. The writeback problem for this workload is trivially parallelizable, but that is not always true. The key problem is how to get the data out of the page cache and to the disk as efficiently as possible. 

Jan Kara said that it would be difficult to add more writeback threads because there are assumptions that there is only one at various levels. He and Chinner discussed ways to do so, though it sounds like there would be quite a bit of work. Part of the reason it has not really been investigated, perhaps, is that SSDs are so fast that there is less of a push to optimize these kinds of things, Ts'o said. Getting a, say, 20% benefit on an untar-and-build workload, which already runs quickly, is not all that compelling. 

There may be opportunities to simply turn off writeback on certain classes of devices, since writethrough performs so much better on high-end SSDs, Ts'o said. Chamberlain wondered if switching to writethrough would help solve the buffered I/O atomic-write problem; Chinner said that it could. With that, the session ran out of time, though there was talk of picking it back up at BoF session later in the summit.
