---
title: Debating overlayfs
url: https://lwn.net/Articles/447650/
date: "June 15, 2011"
category: "Filesystems-Union; Overlayfs"
author: "By Jonathan Corbet June 15, 2011"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
June 15, 2011

Union filesystems allow multiple filesystems to be combined and presented to the user as a single tree. In typical use, a writable filesystem is overlaid on top of a read-only base, creating the illusion that all files on the filesystem can be changed. This mode of operation is useful for live CD distributions, embedded systems where a quick "factory reset" capability is desired, virtualized systems built on a common base filesystem, and more. Despite the value of this feature, Linux has never had an in-kernel union filesystem option, despite several attempts to create one. A recent attempt to change that situation may or may not succeed. 

LWN [looked at the overlayfs filesystem](<https://lwn.net/Articles/403012/>) last year. Overlayfs, written by Miklos Szeredi, is distinguished by its relative simplicity. Recently, Miklos [asked](<https://lwn.net/Articles/447654/>) if overlayfs could be merged for the 3.1 development cycle. He may get his wish, but some worries will have to be addressed first. 

Andrew Morton has raised a couple of concerns; [one of which](<https://lwn.net/Articles/447657/>) is that the problem might be better solved in user space. He dismissed the simplicity of overlayfs, saying ""Not merging it would be even smaller and simpler,"" and suggested that performance problems should be addressed by making the user-space implementation faster. Linus has pretty much [ended that aspect of the debate](<https://lwn.net/Articles/447658/>) by saying ""People who think that userspace filesystems are realistic for anything but toys are just misguided."" So the way seems to be clear for a union filesystem implementation in the kernel. 

Andrew's other [concern](<https://lwn.net/Articles/447659/>) is that overlayfs may not be a sufficiently complete solution: 

If overlayfs doesn't appreciably decrease the motivation to merge other unioned filesystems then we might end up with two similar-looking things. And, I assume, the later and more fully-blown implementation might make overlayfs obsolete but by that time it will be hard to remove. 

That objection is harder to answer. It has been pointed out that [OpenWRT is happily using overlayfs](<https://lwn.net/Articles/447660/>) and [Ubuntu is considering it](<https://lwn.net/Articles/447661/>). About the only viable alternative project is [union mounts](<https://lwn.net/Articles/396020/>), which has not seen much developer attention recently. On the feature front, it doesn't seem like anything else will come along and outshine overlayfs in the near future. 

On the technical side, union filesystems have always presented some unique challenges. Valerie Aurora, who has done a fair amount of work in this area, [looked at overlayfs](<https://lwn.net/Articles/435019/>) in March and seemed to be positive about it: 

I took a quick look at the current overlayfs patch set, and it's small, clean, and easy to understand. If it does what people need, I say ship it. 

She has [changed her tune](<https://lwn.net/Articles/447666/>) a bit in the current discussion, suggesting that there are some difficulties which need to be addressed: 

Overlayfs is not the simplest possible solution at present. For example, it currently does not prevent modification of the underlying file system directories, which is absolutely required to prevent bugs according to Al. Al proposed a solution he was happy with (read-only superblocks), I implemented it for union mounts, and I believe it can be ported to overlayfs. But that should happen *before* merging. 

She raised some locking concerns as well, which Miklos [addressed in detail](<https://lwn.net/Articles/447671/>); the concern about changing the underlying filesystem has not been answered, though. So it's possible that technical correctness issues may yet delay the merging of overlayfs into the kernel. That said, it seems clear that there is demand for this feature, and that overlayfs appears to satisfy that demand nicely. There will likely come a time when keeping it out of the kernel becomes too hard to justify.
