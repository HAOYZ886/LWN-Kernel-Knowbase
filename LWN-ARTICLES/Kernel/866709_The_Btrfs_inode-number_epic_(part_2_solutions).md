---
title: "The Btrfs inode-number epic (part 2: solutions)"
url: https://lwn.net/Articles/866709/
date: "August 23, 2021"
category: "Filesystems-Btrfs"
author: "By Jonathan Corbet August 23, 2021"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
August 23, 2021

[The first installment](<https://lwn.net/Articles/866582/>) in this two-part series looked at the difficulties that arise when Btrfs filesystems containing subvolumes are exported via NFS. Btrfs has a couple of quirks that complicate life in this situation: the use of separate device numbers for subvolumes and the lack of unique inode numbers across the filesystem as a whole. Recently, Neil Brown set off on an effort to try to solve these problems, only to discover that the situation was even more difficult than expected and that many attempts would be required. 

#### Take 1: internal mounts

Brown's [first patch set](<https://lwn.net/ml/linux-fsdevel/162742539595.32498.13687924366155737575.stgit@noble.brown/>) attempted to resolve these problems by creating the concept of "internal mounts"; these would be automatically created by Btrfs for each visible subvolume. The automount mechanism is used to make those mounts appear. With a bit of tweaking, the kernel's NFS server daemon (`nfsd`) can recognize these special mounts and allow them to be crossed without an explicit mount on the remote side. With this setup, the device numbers shown by the system are as expected, and inode numbers are once again unique within a given mount. 

At a first glance, this patch set seemed like a good solution to the problem. When presented with a description of this approach back in July, filesystem developer Christoph Hellwig [responded](<https://lore.kernel.org/linux-nfs/YPaCuGm+3RX6vzjp@infradead.org/>): ""This is what I've been asking for for years"". With these changes, Btrfs appears to be a bit less weird and some longstanding problems are finally resolved. 

This patch set quickly ran into trouble, though. Al Viro [pointed out](<https://lwn.net/ml/linux-fsdevel/YQNG+ivSssWNmY9O@zeniv-ca.linux.org.uk/>) that the mechanism for querying device numbers could generate I/O while holding a lock that does not allow for such actions, thus deadlocking the system; without that query, though, the scheme for getting the device number from the filesystem will not work. One potential alternative, providing a separate superblock for each internal mount that would contain the needed information, is even worse. Many operations in the kernel's virtual filesystem layer involve iterating through the full list of mounted superblocks; adding thousands of them for Btrfs subvolumes would create a number of new performance problems that would take massive changes to fix. 

Additionally, Amir Goldstein [noted](<https://lwn.net/ml/linux-fsdevel/CAOQ4uxj9DW2SHqWCMXy4oRdazbODMhtWeyvNsKJm__0fuuspyQ@mail.gmail.com/>) that the new mount structure could create trouble for overlayfs; it would also break some of his user-space tools. There is also the little issue of how all those internal mounts would show up in `/proc/mounts`; on systems with large numbers of subvolumes, that would turn `/proc/mounts` into a huge, unwieldy mess that could also expose the names of otherwise private subvolumes. 

#### Take 2: file handles

Brown [concluded](<https://lwn.net/ml/linux-fsdevel/162787790940.32159.14588617595952736785@noble.neil.brown.name/>) that ""with its current framing the problem is unsolvable"". Specifically, the problem is the 64 bits set aside for the inode number, which are not enough for Btrfs even now. The problem gets worse with overlayfs, which must combine inode numbers from multiple filesystems, yielding something that is necessarily larger than any one filesystem's numbers. Brown described the current solution in overlayfs as ""it over-loads the high bits and hopes the filesystem doesn't use them"", which seems less than fully ideal. But, as long as inode numbers are limited to any fixed size, there is no way around the problem, he said. 

It would be better, he continued, to use the file handle provided by many filesystems, primarily for use with NFS; a file's handle can be obtained with [`name_to_handle_at()`](<https://man7.org/linux/man-pages/man2/open_by_handle_at.2.html>). The handle is of arbitrary length, and it includes a generation number, which handily gets around the problems of inode-number reuse when a file is deleted. If user space were to use handles rather than inode numbers to check whether two files are the same, a lot of problems would go away. 

Of course, some new problems would also materialize, mostly in the form of the need to make a lot of changes to user-space interfaces and programs. No files exported by the kernel (`/proc` files, for example) use handles now, so a set of new files that included the handles would have to be created. Any program that looks at inode numbers would have to be updated. The result would be a lot of broken user-space tools. Brown has repeatedly [insisted](<https://lwn.net/ml/linux-fsdevel/162794157037.32159.9608382458264702109@noble.neil.brown.name/>) that breaking things may be possible (and necessary): 

> If you refuse to risk breaking anything, then you cannot make progress. Providing people can choose when things break, and have advanced warning, they often cope remarkable well. 

Incompatible changes remain a hard sell, though. Beyond that, to get the full benefit from the change, Btrfs would have to be changed to stop using artificial device numbers for subvolumes, which is not a small change either. And, as Viro [pointed out](<https://lwn.net/ml/linux-fsdevel/YQeB3ASDyO0wSgL4@zeniv-ca.linux.org.uk/>), it is possible for two different file handles to refer to the same file. 

In summary, this approach did not win the day either. 

#### Take 3: mount options

Brown's [third attempt](<https://lwn.net/ml/linux-fsdevel/162848123483.25823.15844774651164477866.stgit@noble.brown/>) approached the problem from a different direction, making all of the changes explicitly opt-in. Specifically, he added two new mount options for Btrfs filesystems that would change their behavior with regard to inode and device numbers. 

The first option, `inumbits=`, changes how inode numbers are presented; the default value of zero causes the internal object ID to be used (as is currently the case for Btrfs). A non-zero value tells Btrfs to generate inode numbers that are ""mostly unique"" and that fit into the given number of bits. Specifically, to generate the inode number for a given object within a subvolume, Btrfs will: 

  * Generate an "overlay" value from the subvolume number; this is done by byte-swapping the number so that the low-order bits (which vary the most between subvolumes) are in the most-significant bit positions. 
  * The overlay is right-shifted to fit within the number of bits specified by `inumbits=`. If that number is 64, no shift need be done. 
  * That overlay value is then XORed with the object number to produce the inode number presented to user space. 

The resulting inode numbers will still be unique within any given subvolume; collisions within a large Btrfs filesystem can still happen, but they are less likely than before. Setting `inumbits=64` minimizes the chances of duplicate inode numbers, but a lower number (such as 56) may make sense in situations (such as when overlayfs is in use) where the top bits are used by other subsystems. 

The second mount option is `numdevs=`; it controls how many device numbers are used to represent subvolumes within the filesystem. The default value, `numdevs=many`, preserves the existing behavior of allocating a separate device number for every subvolume. Setting `numdevs=1`, instead, causes a single device number to be used for all subvolumes. When a filesystem is mounted with this option, tools like `find` and `du` will not be able to detect the crossing of a subvolume boundary, so their options to stay within a single filesystem may not work as expected. It is also possible to specify `numdevs=2`, which causes two device numbers to be used in an alternating manner when moving from one subvolume to the next; this makes tools like `find` work as expected. 

Finally, this patch set also added the concept of a "tree ID" that can be fetched with the [`statx()` system call](<https://man7.org/linux/man-pages/man2/statx.2.html>). Btrfs would respond to that query with the subvolume ID, which applications could then use to reliably determine whether two files are contained within the same subvolume or not. 

Btrfs developer Josef Bacik [described](<https://lwn.net/ml/linux-fsdevel/e6496956-0df3-6232-eecb-5209b28ca790@toxicpanda.com/>) this work as ""a step in the right direction"", but said that he wants to see a solution that does not require special mount options. ""Mount options are messy, and are just going to lead to distros turning them on without understanding what's going on and then we have to support them forever"". A proper solution, he said, does not present the possibility for users to make bad decisions. He suggested just using the new tree ID within `nfsd` to solve the NFS-specific problems, generating new inode numbers itself if need be. 

Brown [countered](<https://lwn.net/ml/linux-fsdevel/162872000356.22261.854151210687377005@noble.neil.brown.name/>) with a suggestion that, rather than adding mount options, he could just create a new filesystem type (""btrfs2 or maybe betrfs"") that would use the new semantics. Bacik [didn't like that idea](<https://lwn.net/ml/linux-fsdevel/6571d3fb-34ea-0f22-4fbe-995e5568e044@toxicpanda.com/>) either, though. Brown [added](<https://lwn.net/ml/linux-fsdevel/162880775121.15074.3288255136681201159@noble.neil.brown.name/>) that he would prefer not to do ""magic transformations"" of Btrfs inode numbers in `nfsd`; if a filesystem requires such operations, they should be done in the filesystem itself, he said. He then asked that the Btrfs developers make a decision on their preferred way to solve this problem, but did not get an answer. 

#### Take 4: the uniquifier

On August 13, Brown returned with [a minimal patch](<https://lwn.net/ml/linux-fsdevel/162881913686.1695.12479588032010502384@noble.neil.brown.name/>) aimed at solving the NFS problems that started this whole discussion. It enables a filesystem to provide a "uniquifier" value associated with a file; this value, the name of which is arguably better suited to a professional wrestler, is only available within the kernel. The NFS server can then XOR that value with the file's inode number to obtain a number that is more likely to be unique. Btrfs provides the overlay value described above as this value; `nfsd` uses it, and the problem (mostly) goes away. 

Bacik [said](<https://lwn.net/ml/linux-fsdevel/ebffa324-0d42-4227-b162-0dad19144943@toxicpanda.com/>) that this approach was ""reasonable"" and acked it for the Btrfs filesystem. It thus looks like it could finally be a solution for the problem at hand. Or, at least, it's closer; Brown later [realized](<https://lwn.net/ml/linux-fsdevel/162906585094.1695.15815972140753474778@noble.neil.brown.name/>) that the changed inode numbers would create the dreaded "stale file handle" errors on existing clients when the transition happens. An [updated version of the patch set](<https://lwn.net/ml/linux-fsdevel/162969155423.9892.18322100025025288277@noble.neil.brown.name/>) adds a new flag in an unused byte of the NFS file handle to mark "new-style" inode numbers and prevent this error from occurring. 

The second revision of the fourth attempt may indeed be the charm that makes some NFS-specific problems go away for Btrfs users. It is hard not to see this change — an internal process involving magic numbers that still is not guaranteed to create unique inode numbers — as a bit of a hack, though. Indeed, even Brown [referred to](<https://lwn.net/ml/linux-fsdevel/162942971499.9892.4386273250573040668@noble.neil.brown.name/>) ""hacks to work around misfeatures in filesystems"" when talking about this work. Hacks, though, can be part of life when dealing with production systems and large user bases; a cleaner and more correct solution may not be possible without breaking user systems. So the uniquifier may be as good as things can get until some other problem is severe enough to force the acceptance of a more disruptive solution.
