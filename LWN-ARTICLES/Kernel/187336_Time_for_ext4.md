---
title: "Time for ext4?"
url: https://lwn.net/Articles/187336/
date: "June 12, 2006"
category: "Development model; Filesystems-ext3"
author: 
---

As described in [this article](<https://lwn.net/Articles/187321/>), patches which add extents and 48-bit capability to the ext3 filesystem have been circulated for wider review. Everybody seems to agree that these changes are good and should be part of the Linux kernel. Well, [almost everybody](<https://lwn.net/Articles/187337/>) agrees. But the way in which these features get in has become the inspiration for an extended discussion on how filesystem development should work. 

Some developers, most prominently Jeff Garzik, have [expressed concerns](<https://lwn.net/Articles/187339/>) about merging these changes into ext3; they would rather see a new ext4 filesystem created for new features. There are a number of reasons put forward for doing things this way. First and foremost, perhaps, is the fact that using the extents/48-bit features results in filesystems which are no longer backward compatible. If a system administrator enables extents on a filesystem, a special "incompatible feature" flag will be set in the filesystem superblock. Thereafter, it will no longer be possible to mount that filesystem with any older kernel which does not recognize that flag. Until now, it has generally been possible to mount ext3 filesystems on older kernels - even those which only support ext2 (with one ugly exception involving a distributor which was heavily pushing SELinux features). 

The overall effect of all these changes on filesystem stability is also a concern. Filesystems are important, and users tend to take a very dim view of "upgrades" which introduce bugs or impact performance. As [Linus puts it](<https://lwn.net/Articles/187340/>): 

For me, the biggest cost tends to actually be support. A stable filesystem that is used by thousands and thousands of people and that isn't actually developed outside of just maintaining it IS A REALLY GOOD THING TO HAVE. 

The incorporation of major features into ext3 certainly takes it out of the "just maintaining it" realm. 

As more features are added, the filesystem code (which must support filesystems both with and without that feature) gets more complicated. In particular, one sees increasing amounts of code which looks like: 

```
if (has_this_fancy_feature)
        	do_it_the_fancy_way();
        else
        	do_it_the_old_boring_way();
```

Such code can be harder to follow, and it tends not to isolate the new feature code as nicely as one might like. If, instead, one were to put the new features into a new filesystem, a lot of these conditionals could be taken out. 

Finally, it is said that the need to be so careful about backward compatibility is a drag on filesystem development. By separating development filesystems from those which are meant to be stable, the developers can push forward with the new capabilities they would like to implement. For practical examples, consider the separation of ext2 and ext3, the separation of the SMB and CIFS filesystems, and the creation of libata rather than shoehorning serial ATA support into the old ATA drivers. 

Needless to say, the ext3 developers have their own take on all of this. A filesystem with the new features will not work on older kernels regardless of whether it is called ext3 or ext4. Since a feature like extents must be explicitly enabled by the system administrator (assuming the distributor does not quietly do it for them), nobody should be surprised by a filesystem which no longer works on older systems. Pushing the new features into an ext4 would simply slow their uptake without buying much else. 

While some think that splitting out development into a new filesystem will ease code maintenance, others are less sure. In particular, there is worry that bugs fixed in one of the filesystems may not get fixed in the other. 

It has been noted, repeatedly, that users very much like to be able to get new features into their filesystems without having to backup and restore the whole thing. The transition from ext2 to ext3 is a clear example of how this can work; if moving to ext3 had required restoring the filesystem from scratch, ext3 would have been adopted much more slowly, and less universally, than it was. As this example shows, however, putting new features into a new ext4 filesystem would not necessarily preclude this sort of upgrade. 

The ext3 developers also point out that they have been working on that filesystem for many years and have not yet created big problems for the Linux user base. They have, they feel, earned a certain amount of trust. So they would rather move ahead with some features which have been put together with great care and extensive review rather than cloning ext3 into ext4 and starting something new. 

An attempt to guess how all this might settle out could start with [these words from Linus](<https://lwn.net/Articles/187349/>): 

Quite frankly, at this point, there's no way in hell I believe we can do major surgery on ext3. It's the main filesystem for a lot of users, and it's just not worth the instability worries unless it's something very obviously transparent. 

[Yet another point of view](<https://lwn.net/Articles/187346/>) comes from Andrew Morton: 

All that being said, Linux's filesystems are looking increasingly crufty and we are getting to the time where we would benefit from a greenfield start-a-new-one. That new one might even be based on reiser4 - has anyone looked? It's been sitting around for a couple of years. 

As reiser4 shows, getting a truly new filesystem into the kernel isn't necessarily an easy thing to do. It may well not happen before large numbers of users start running into the current limits of ext3. So the current set of enhancements will probably find its way in - though what the resulting filesystem will be called is still not entirely clear.
