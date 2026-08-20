---
title: CoreOS looks to move from Btrfs to overlayfs
url: https://lwn.net/Articles/627232/
date: "December 24, 2014"
category: "Btrfs; Filesystems-Btrfs; Filesystems-Union"
author: "By Jake Edge December 24, 2014"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
December 24, 2014

After many years of different union filesystem implementations trying to get into the mainline, the [overlay filesystem](<https://lwn.net/Articles/403012/>) (also known as overlayfs) was finally merged for 3.18. It didn't take all that long for at least one project to notice and react to that addition. [CoreOS](<https://coreos.com/>), which is a [Linux distribution for large server deployments](<https://lwn.net/Articles/593928/>), is now planning to move its root filesystem images from Btrfs to ext4—with overlayfs on top. 

Various filesystem features are used by [Docker](<https://www.docker.com/>) (which is the mechanism used to run applications on CoreOS—at least [for now](<https://lwn.net/Articles/624349/>)) to put read-write filesystems atop the read-only base that provides the root filesystem. In addition, Docker applications each have their own read-only filesystem layer that is currently typically handled in CoreOS by using Btrfs's copy-on-write and snapshot features. Moving to a union filesystem like overlayfs will provide the same basic functionality, just using different underlying techniques. 

Brandon Phillips [proposed the switch](<https://lwn.net/Articles/627339/>) to the coreos-dev mailing list on December 15 and the reaction has generally been quite positive. Btrfs is, it seems, still a bit immature. As Phillips noted: ""CoreOS users have regularly reported bugs against btrfs including: out of disk space errors, metadata rebalancing problems requiring manual intervention and generally slow performance when compared to other filesystems"". 

That proposal was greeted by responses from several others who had seen the problems that Phillips mentioned. Seán C. McCord [pointed out](<https://lwn.net/Articles/627356/>) that he is a Btrfs proponent, but would still be happier using ext4 and overlayfs: 

The out-of-space / metadata balancing problem has bitten me more times than I care to count. It's essentially a fact of life that I have to blow away /var/lib/docker and all its subvolumes every few weeks on any given machine, to clear an out-of-space problem (though `df` shows a usage of, say, 30%). 

But, in the only real opposition seen in the thread, Morgaine Fowle [noted his excitement](<https://lwn.net/Articles/627387/>) about the features that Btrfs brings to the table and thinks CoreOS should be focusing on those, rather than what he sees as a cobbled-together solution using overlayfs. Furthermore: 

I deeply enjoy the file-system taking responsibility for snapshotting. It creates a consistent management interface that's useful for a wide range of tasks. Anything based off overlayfs is going to have to concoct it's own unique management layer which will require it's own domain knowledge to handle, where-as someone proficient with the filesystem's snapshotting tools is going to have a more general, portable knowledge they'll be able to use to make sense of what CoreOS is doing naturally. 

But, according to Phillips's proposal, overlayfs will bring some benefits beyond just more stability. He pointed to a [Red Hat evaluation of storage options for Docker](<http://developerblog.redhat.com/2014/09/30/overview-storage-scalability-docker/>) that showed overlayfs as a faster choice for creating and destroying containers. In addition, it also said that overlayfs uses memory more efficiently since it can keep a single copy of a file's pages in the page cache, which can then be used by multiple containers. Since there tends to be a lot of overlap between containers, this can result in significant performance improvements. There are some downsides to overlayfs, too, of course, including that changes to files in the underlying read-only layer requires a potentially costly copy-up operation. 

Btrfs creator Chris Mason also [posted](<https://lwn.net/Articles/627412/>) to the thread. He noted that a number of the problems (""warts"") that CoreOS users were running into are being addressed: 

The 3.19 merge window fixes some very hard to find corruption problems that we've been chasing down, and Josef Bacik has developed a slick power-fail testing target that makes it much easier to prevent similar bugs in the future. 3.19 will also fix rare corruptions with block group removal, making both balance and the new auto-blockgroup cleanup feature much more reliable. 

Overall, though, Mason was not particularly disappointed or unhappy about the proposal to switch to overlayfs, saying that CoreOS should choose the storage solution that best fits its needs. He was also glad to see projects looking to adopt overlayfs now that it has been added to the kernel. Similarly, Greg Kroah-Hartman [congratulated CoreOS](<https://plus.google.com/u/0/+gregkroahhartman/posts/ExT46iXqL3K>) for using overlayfs in a post to Google+. 

The main change outlined by Phillips would be to move the root filesystem images from Btrfs to ext4. Eventually, the [Docker overlayfs graph backend](<https://github.com/docker/docker/pull/7619>) would be made the default, but existing Btrfs-based CoreOS systems would continue to work as they are. Given that there were almost no complaints about the proposal, with multiple posts agreeing (as well as quite a few "+1" posts), it would appear to be the path forward for CoreOS. 

It should be noted that overlayfs itself has only been in the kernel for a short time. The patches been around for quite a while now, and have been used by various distributions along the way, but it probably still has a few bugs that will need to be shaken out. It is far less complex than Btrfs, however, which presumably reduces the risks of switching from one immature storage technology to another. At this point, openSUSE is the only major distribution to have [adopted Btrfs](<https://news.opensuse.org/2014/11/12/what-to-expect-from-btrfs-on-opensuse-13-2/>) as its default filesystem, though others have discussed it. 

One conclusion seems inevitable, though: even after many years of development, Btrfs has not reached a level of functionality, performance, and stability required by some. Mason's message provides some hope that we are getting there, but that has seemingly been true for a while now. When we actually get "there" is still anyone's guess at this point.
