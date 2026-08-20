---
title: Shrinking filesystem caches for dying control groups
url: https://lwn.net/Articles/789640/
date: "May 29, 2019"
category: "Control groups; Memory management-Control groups"
author: "By Jake Edge May 29, 2019 LSFMM"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
May 29, 2019

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2019/>)

In a followup to his [earlier session on dying control groups](<https://lwn.net/Articles/787614/>), Roman Gushchin wanted to talk about problems with the shrinkers and filesystem caches in a combined filesystem and memory-management session at the 2019 Linux Storage, Filesystem, and Memory-Management Summit (LSFMM). Specifically, for control groups that share the same underlying filesystem, the shrinkers are not able to reclaim memory from the VFS caches after a control group dies, at least under slight to moderate memory pressure. He wanted to discuss how to reclaim that memory without major performance impacts. 

The starting point might be to determine how to calculate the memory pressure to apply, he said. Back in October and November, there were several proposals on doing that; his [patch](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=172b06c32b949759fe6313abec514bc4f15014f4>) was reverted due to performance regressions, but there were others, none of which went upstream. 

[ ![\[Roman Gushchin\]](https://static.lwn.net/images/2019/lsf-gushchin-sm.jpg) ](<https://lwn.net/Articles/789645/>)

Chris Mason asked if there was a need to reparent the caches. Gushchin said that was already being done, but that there is no way to move pages between different caches, so references to shared objects persist. Christoph Lameter suggested making slab objects movable, so that things like directory entry (dentry) cache entries and inodes could be moved, but Gushchin said that the objects are in use, so they cannot be moved. Lameter said that it would take some work, but those objects could be made movable. 

James Bottomley said that he didn't think this was a shrinker problem, exactly. The objects are still in use based on the reference counts so they should not be reclaimed. Gushchin said that the current shrinker implementation tries to minimize the number of objects it has to scan, so unless there is major memory pressure, it doesn't scan anything. Small objects held by a dying control group could be holding onto a large amount of memory, but when calculating the pressure, the system does not know if that is the case. 

Bottomley said that the idea behind the shrinkers is to reclaim just the amount of memory needed, not to reclaim it all. So if you think you have 100MB of reclaimable memory, but only need ten pages, that's all the shrinkers are meant to give you. Changing that will cause regressions in lots of other places. 

What was proposed, Gushchin said, was to provide additional pressure so that some amount of scanning is done. Right now, the shrinkers don't scan anything, then the system runs out of memory, so all of the reclaimable memory gets freed. Bottomley said that perhaps the solution is not in the shrinkers, but in handling the dying control groups differently. 

The problem is that the kernel cannot shrink hard enough without impacting performance, Mason said. It is the same problem that was discussed in the earlier session; there needs to be a way to move or copy the objects elsewhere so the dying control group no longer owns the memory. Gushchin said that he didn't know if trying to move pages between caches is "totally crazy" or not. After a long pause, Mason said "I think it should be easy" to laughter. 

An attendee asked if control groups really need to use different pages or if their pages could be merged by the allocator. Gushchin said that control groups are charged for memory on a per-page basis; each page belongs to a particular control group. Bottomley summarized the problem by saying that if three objects are allocated normally, they all likely end up in the same page, but if they are allocated by three different control groups, they each end up in their own page. Another attendee noted that once a control group goes away, the page with that object will not be filled further and may not be reclaimed for some time. 

That led Bottomley to wonder if the page's ownership could be switched to a different control group; that way the memory references to the object would not have to change. Matthew Wilcox rephrased that as donating the slab page to another similar slab in the system that is associated with a still-running control group. Ted Ts'o said there is a policy question with that approach, as suddenly a control group gets charged with a new page. But Wilcox stressed the word "donate"; the new control group would not be taxed for the new page. "No taxation without allocation", he said, to groans and chuckles. 

There was some discussion of switching to a per-byte charging model for control groups, but the complexity seemed high. Bottomley said that any attempt to change the charging policy would be reopening a "big can of worms". After that, Mason asked Gushchin if the discussion had made things easier or harder. Gushchin said that it was "hard to say", there are several different kinds of objects that come into play. 

Mason said that the most complicated thing to move would be the inodes because there are lots of pointers from pages back to the inode. There may well be other slabs that are far worse that he doesn't know about, however. Lameter said that making these objects movable would solve a lot of problems and not just for this particular situation. Making objects that are frequently allocated, such as dentries and inodes, movable would be an overall improvement to the kernel. It would, for example, make it easier to assemble huge pages when needed. 

Ts'o asked if anyone had looked to see which slab objects are the most problematic. His guess would be the inodes, which is also the hardest to deal with. But, if so, it might also give the "most bang for the buck". Gushchin said it is mostly dentries and inodes. Mason said that inodes require the most I/O to get them back so it would be worth preserving them if possible. 

If the pages were donated to some common cache, the next allocation of that size that required a new page could return the partly filled page, an attendee said. It would be more efficient than donating it to another control group when it is not known that the group will actually need to do more allocation. Wilcox called that a kind of "lazy donation". Ts'o added that donating a page that contained, say, an inode owned by a dying control group would at least allow the rest of the page to be used by someone, rather than just wasting most of a page. 

The problem with donating cache pages is that there is no way to get from a control group to the list of slab pages that it has objects in, Mason said. From a complexity point of view, it needs to be determined if it is worth tracking that and keeping it up to date. At that point, the discussion trailed off without any real resolution other than some possible paths forward.
