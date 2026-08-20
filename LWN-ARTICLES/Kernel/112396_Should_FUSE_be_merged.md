---
title: "Should FUSE be merged?"
url: https://lwn.net/Articles/112396/
date: "November 23, 2004"
category: "Filesystems-In user space"
author: 
---

The [Filesystems in User Space (FUSE) patch](<https://lwn.net/Articles/68104/>) has been around for some time. FUSE acts as a kernel filesystem which turns around and passes all VFS requests out to a user-space daemon, which is expected to do something reasonable with them. There are [numerous projects using FUSE](<http://fuse.sourceforge.net/filesystems.html>) to implement interesting filesystems in user space. The FUSE developers have now [requested that FUSE be merged](<https://lwn.net/Articles/112397/>) into the 2.6 kernel. They may yet get there, but some obstacles stand in the way. 

Linus started by [complaining](<https://lwn.net/Articles/112399/>) that FUSE was "too messy." Some of his impressions, it turns out, may have been based on a reading of old code. Some of the things he was complaining about were parts of the 2.4 version of the patch; they are not present in the version being put forward for inclusion. 

There is, however, one show-stopping problem which remains in the code. If the system's memory gets to be full of dirty pages which must be written to a FUSE filesystem, and the user-space process which implements that filesystem has been swapped out, the system can deadlock. It cannot clean up those dirty pages until they have been written to the backing store, it cannot write those pages until the user-space daemon has been paged in, and it cannot page in the daemon until the dirty pages are cleaned. The system comes to a screeching halt and the users reconsider the whole idea of user-space filesystems. 

The problem is most easily demonstrated through the use of shared writable mappings. With such mappings, user space can create vast numbers of dirty pages without the operating system knowing about it. Andrew Morton [demonstrated](<https://lwn.net/Articles/112413/>) that this is not just a theoretical problem; it can be made to happen on real systems. The problem can also be made to happen by simply writing too much data to the filesystem. All this led Linus to [lecture](<https://lwn.net/Articles/112414/>) on the topic: 

Guys, there is a _reason_ why microkernels suck. This is an example of how things are _not_ "independent". The filesystems depend on the VM, and the VM depends on the filesystem. You can't just split them up as if they were two separate things (or rather: you _can_ split them up, but they still very much need to know about each other in very intimate ways). 

In this case, the worst problems can be avoided by simply disallowing shared, writable mappings. That limitation will not, in fact, bother too many people; these mappings are not heavily used. It's also necessary to take steps like limiting the number of pages currently queued for writing out. This limit will affect users, in that it will reduce performance. It has been noted, however, that deadlocks tend to have an even worse impact on performance. 

In response to the above concerns (and others), the FUSE patches [have been reworked](<https://lwn.net/Articles/112199/>). Among other things, the shared, writable mapping support has been split out into a separate, optional patch. There's no word on whether it will be merged, though Linus did [suggest that it might](<https://lwn.net/Articles/112415/>): 

I'm a sucker. Ask anybody. I'll accept the exact same patch that I rejected earlier if you just do it the right way. I'm convinced that some people actually do it on purpose just for the amusement value ("Look, he did it _again_. What a doofus!") 

Whether Andrew Morton is so gullible remains to be seen.
