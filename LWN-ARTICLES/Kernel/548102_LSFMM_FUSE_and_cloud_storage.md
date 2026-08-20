---
title: "LSFMM: FUSE and cloud storage"
url: https://lwn.net/Articles/548102/
date: "April 24, 2013"
category: "Filesystems-Ceph; Filesystems-In user space"
author: "By Jake Edge April 24, 2013 LSFMM Summit 2013"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
April 24, 2013

* * *

[LSFMM Summit 2013](<https://lwn.net/Articles/LSFMM2013/>)

In a plenary discussion that was ostensibly about cloud storage, most of the time was spent on enhancements to the FUSE (filesystem in user space) subsystem. FUSE is used by some cloud storage filesystems, which is the connection. Most of the session was led by Maxim Patlasov, who has been experimenting with FUSE fixes and upgrades for the last year or so. In that time, he has made six significant improvements to FUSE, he said. 

All of the enhancements are targeted at performance, to bring FUSE-based filesystems closer to the performance of their in-kernel siblings. For example, a change to the writeback cache policy gave [GlusterFS](<http://www.gluster.org/>) a factor of four improvement in sequential 4K writes, he said. It also increased the speed of [S3QL](<http://code.google.com/p/s3ql/>) by a factor of five for sequential 512K writes. 

Similarly, optimizing scatter/gather direct I/O greatly improved performance. It was a "trivial change", but GlusterFS saw a factor of 15 improvement when combining 256 4K buffers into a single I/O operation. Essentially, the filesystem only submits one request to user space, rather than many. Another change to process direct I/O requests asynchronously increased performance for GlusterFS rewrites, Patlasov said. 

He has been experimenting on using FUSE with in-kernel direct I/O, which is a clear, easy-to-understand model for doing I/O. Using asynchronous direct I/O to pass a `struct bio` vector to the block device from the in-kernel FUSE server provides excellent performance, he said. 

As he often does, James Bottomley cut to the chase. What Patlasov has shown is that "Gluster sucks", but that 95% of in-kernel performance can be achieved with a FUSE-based filesystem provided that the [in-kernel direct asynchronous I/O patches](<https://lwn.net/Articles/534928/>) are merged, Bottomley said. 

Bottomley asked Al Viro if he had any objections to those patches. Viro said that he had "no objections right now", but that was only because he hadn't looked at them yet. Objections "will appear" as soon as he reads through them, he said. Bottomley noted that one action item should be to hopefully get Viro to do that review soon. 

After that was a discussion of the problem of accounting for dirty pages in FUSE filesystems that eventually was spun off into a memory management track [discussion of its own](<https://lwn.net/Articles/548108/#fuse-wb>). 

Right after that discussion, Andrew Morton stood up to remind everyone to do a better job of copying interested parties on patches. He noted that both the patch proposed to solve the FUSE dirty page accounting problem and the one to do in-kernel direct I/O had been posted to the linux-kernel mailing list, but not copied to him (and perhaps others who would be interested). Since there are few who are able to read the entire list, and even those who do largely skim the subject lines, it is important to find the right people to CC on a patch. 

With that, the patiently waiting Sage Weil took over the discussion lead role to talk about issues for the [Ceph](<http://ceph.com/>) distributed filesystem. He too had issues with FUSE, but said that most of those were getting resolved. The biggest issue right now was "everybody's favorite subject", controlling writeback. Right now, Ceph does a bunch of hacks to try to force writeback to happen, but he would like to do something more straightforward. 

One possibility is to use a memory control group (memcg) and set the parameters on the group such that the data gets pushed out. Jan Kara warned that memcg is not going to work for what Weil is trying to do. In theory, you could set per-`bdi` limits, timeouts, and other tunables, that would allow Weil to do that, but it is functionality that would need to be added, Kara said. 

There are two other things Weil is looking at for Ceph. The first is to use `splice()` to eliminate copies between user and kernel space, but Samba developers say that the performance of `splice()` "sucks", so he will need to look into that. The other potential new feature is end-to-end checksums for data. It is a feature that Btrfs already has, so it would be better to have a more unified solution for both (and, eventually, others).
