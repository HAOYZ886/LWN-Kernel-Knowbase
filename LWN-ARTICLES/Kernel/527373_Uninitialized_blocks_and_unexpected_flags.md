---
title: Uninitialized blocks and unexpected flags
url: https://lwn.net/Articles/527373/
date: "November 28, 2012"
category: "Development model-Code review; Filesystems-ext4; fallocate"
author: "By Jonathan Corbet November 28, 2012"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
November 28, 2012

One often-heard complaint in the early BitKeeper era was that, by letting code reach the mainline without going via a mailing list, BitKeeper made it easy for maintainers to slip surprise changes in underneath the review radar. Those worries have mostly proved unfounded; when surprises have happened, the response from the community has usually helped to ensure that there would be no repeats. But some developers are charging that the 3.7 kernel contains exactly this type of stealth change and are demanding that it be reverted. 

#### Background

The `fallocate()` system call is meant to be a way for an application to request the efficient allocation of blocks for a file. Use of `fallocate()` allows a process to verify that the required disk space is available, helps the filesystem to allocate all of the space in a single, contiguous group, and avoids the overhead that block-by-block allocation would incur. In the absence of an `fallocate()` implementation (each filesystem must implement it independently), the C library will emulate it by simply writing zeroes to the requested block range; that gets the space allocated, but is less efficient than one would like. The implementation of `fallocate()` within filesystems tries to be more efficient than that; one way to do so is to avoid the process of writing zeroes to the newly-allocated blocks. 

Leaving stale data in allocated blocks has obvious security implications: a hostile application could read those blocks in the hopes of finding confidential documents, passwords, or the missing Fedora 18 Beta release announcement. To avoid this exposure, filesystems like ext4 will mark unwritten blocks as being uninitialized; any attempt to read those blocks will be intercepted and just return zeroes. In the normal case, the application will write data to those blocks before ever trying to read them; writing obviously initializes the blocks without the need to write zeroes first. This implementation seems like it should be about optimal. 

Except that, seemingly, ext4 marks uninitialized blocks at the extent (group of contiguous blocks) level. So, if an application writes to one uninitialized block, the containing extent must be split and the newly-written block(s) added to the previous extent, if possible. That turns out to be more expensive than some users would like. So a shortcut was attempted. 

That shortcut first appeared in April, 2012, in the form of [a new `fallocate()` flag](<https://lwn.net/Articles/492959/>) called `FALLOC_FL_NO_HIDE_STALE`. If `fallocate()` was called with that flag, the newly-allocated blocks would be marked as being initialized even though the old data remained untouched. That obviously brings the old security issues back; to mitigate the problem, the patch added a mount option making the new functionality available only to members of a specific group. That was deemed to be enough, especially for settings where access to the machine as a whole is tightly controlled. 

At least, the authors and supporters of the patch deemed the group check to be enough. The patch was roundly criticized by other filesystem developers; the prevailing opinion appeared to be that it was trying to open up a huge security hole in order to avoid fixing an ext4 performance problem. After that discussion, the patch went away and wasn't heard from again. 

#### A surprise flag

At least, it was not heard from until recently, when some filesystem developers were surprised to discover [this commit](<http://git.kernel.org/linus/bbdd68086ca4a8976226e23efd08e2058d34dd81>) by Ted Ts'o which found its way into the mainline (via the ext4 tree) during the 3.7 merge window. The patch is small and simple; it simply defines the `FALLOC_FL_NO_HIDE_STALE` flag, but adds no code to actually implement it. The changelog reads: 

As discussed at the Plumber's Conference, reserve the bit 0x04 in fallocate() to prevent collisions with a commonly used out-of-tree patch which implements the no-hide-stale feature. 

Filesystem developer Dave Chinner, at least, does not recall this discussion. His response was to post [a patch reverting the change](<https://lwn.net/Articles/527390/>), saying: 

The lack of formal review and discussion for a syscall API change is grounds for reverting patch, especially given the controversial nature of the feature and the previous discussions and NAKs. The way the change was pushed into mainline borders on an abuse of the trust we place in maintainers and hence as a matter of principle this change should be reverted. 

It is true that this particular change is a bit abnormal. It changes the core filesystem code but came by way of a filesystem-specific tree with no acks from any other developers. The patch does not appear to have been posted to any relevant mailing list, violating the rule that all patches should go through public review before being pushed toward the mainline. The addition of a flag with no in-kernel users is also contrary to usual kernel practice. It is, in summary, the sort of change that less well-established kernel developers would never get away with making. So it is hard to fault other filesystem developers for being surprised and unhappy. 

On the other hand, the change just adds a flag definition; it obviously cannot cause problems for existing code. And there does appear to be a real user community for this feature. Ted [justified](<https://lwn.net/Articles/527392/>) his action this way: 

It doesn't change the interface or break anything; it just reserves a bit so that out-of-tree patches don't collide with future allocations. There are significant usages of this bit within Google and Tao Bao. It is true that there has been significant pushback about adding this functionality on linux-fsdevel; I find it personally frustrating that in effect, if enough people scream, they can veto an optional feature that might only be implemented by a single file system. 

This explanation does not appear to have satisfied anybody, though. So we have an impasse of sorts; some developers want a flag to control a functionality they need, while others see it as a security problem and the result of an abuse of the kernel's trust system. 

Alan Cox [suggested](<https://lwn.net/Articles/527394/>) that it would be possible to, instead, reserve a set of filesystem-private flags that could be used for any purpose by any filesystem. Dave [pointed out](<https://lwn.net/Articles/527395/>), however, that a flag bit that behaved differently from one filesystem to the next is a recipe for trouble. His suggestion, instead, is that this functionality should be implemented via the `ioctl()` interface, which is where filesystem-specific options usually hide. The `ioctl()` approach seems like it should be workable, but no patches to that effect have been posted thus far. 

As of this writing, Linus has not accepted the revert, so the `FALLOC_FL_NO_HIDE_STALE` flag can still be found in the 3.7 kernel. He has also remained silent in the discussion. He will have to make a decision one way or the other, though, before the final 3.7 release is made. Once that flag is made available in a stable mainline release, it will be much harder to get rid of, so, if that flag is going to come out, it needs to happen soon.
