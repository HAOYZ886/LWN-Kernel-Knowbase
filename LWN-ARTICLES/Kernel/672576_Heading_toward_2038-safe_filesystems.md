---
title: "Heading toward 2038-safe filesystems"
url: https://lwn.net/Articles/672576/
date: "January 21, 2016"
category: Filesystems; Year 2038 problem
author: "By Jake Edge January 21, 2016"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
January 21, 2016

It is a little hard to call the "year 2038" problem looming, given that it is still nearly 22 years off. But Linux is installed in lots of places where it may continue running past 2038—particularly in embedded systems. Kernel developers have done a fair amount of work to address the problem, much of which we have [covered](<https://lwn.net/Kernel/Index/#Year_2038_problem>) along the way. Attention is now turning to preparing the virtual filesystem (VFS) layer, along with all of the myriad filesystems supported by Linux, for 2038. 

In a nutshell, the problem is that the representation of time on a Linux system—inherited from the original Unix systems—uses a 32-bit signed integer, at least on 32-bit systems. It stores the number of seconds since January 1, 1970, which is known as the "epoch". That value will wrap in January 2038\. The fallout from the [year 2000 problem](<https://en.wikipedia.org/wiki/Year_2000_problem>) was far smaller than expected, but that was largely a user-space issue. The year 2038 problem will affect all existing kernels, so getting ahead of the curve is certainly prudent. 

There are a number of facets to the filesystem side of the problem. Filesystems often store timestamps for each file (Unix filesystems store three), typically in 32-bit formats. That means those filesystems will need to change to a larger-sized timestamp at some point, but they will also need to be able to handle today's already-on-disk filesystems with their 32-bit timestamps. In addition, filesystems may want to handle on-disk timestamps in their own way, without converting to the 64-bit timestamp that is being used internally in the kernel moving forward. 

The VFS layer, on the other hand, has its own timestamp handling for its in-memory inodes and other structures. It will need to change too, but there are various carts and horses that need to be aligned correctly before that can happen. 

Deepa Dinamani recently [posted](<https://lwn.net/Articles/670741/>) a patch set that made an attempt at solving the problem in the VFS layer. Somewhat confusingly to some, it also included patches for some filesystems to try to show the scope of the changes needed. That part of the patch set had not been compile-tested, which was part of the confusion. 

But the first seven (of fifteen) patches targeted VFS. Currently VFS uses a `struct timespec` to represent time. That structure suffers from the year 2038 problem because it uses a `time_t` for seconds, which is 32 bits on some systems. It also uses a `long` to store nanoseconds, which can vary in size as well. That means the structure has a different size on different systems. The replacement for that in a year-2038-compatible world is the `struct timespec64`, which has a 64-bit seconds field, but still has a `long` for nanoseconds, so it still will change size between systems. 

Dinamani proposed using a new `struct inode_timespec` that is defined as a 64-bit seconds field and a 32-bit nanoseconds field everywhere. It is mainly introduced to prevent the need for a big "flag day" patch that converts everything to a `timespec64` at once. She added macros to access the fields so that eventually `inode_timespec` could be turned into a `timespec64`. The `inode_timespec` would be aligned so that it only used 12 bytes, rather than 16 on 64-bit systems. But Dave Chinner [called](<https://lwn.net/Articles/672595/>) that a premature optimization. 

As the conversation continued, there was a clear difference of opinion about how to attack the whole problem. The memory savings for 12 versus 16 bytes for timestamps in inodes in memory may not be that significant, as Arnd Bergmann [pointed out](<https://lwn.net/Articles/672596/>). 32-bit systems will need larger inodes to handle post-2038 timestamps, so it is really a matter of how much they grow. Bergmann copied other architecture mailing lists to see if there were strong feelings about it, but so far there have been no replies. 

But Dinamani also wanted feedback on other parts of the patch set. She [summarized](<https://lwn.net/Articles/672597/>) some of the outstanding questions that needed to be addressed before the problem can be solved. Essentially, there is a tension between the need to move everything to `timespec64` and how that can be done without disrupting filesystem and VFS development. Dinamani sees the transitional `inode_timespec` as something of a necessary evil that will be eliminated once all of the filesystems have been converted. 

Chinner, on the other hand, [thinks](<https://lwn.net/Articles/672598/>) that moving directly to `timespec64` makes more sense. Both agreed that there are some preliminary steps that should be taken, such as ensuring that timestamps are range-checked and clamped to reasonable values on their way into and out of filesystems and VFS. There is also the matter of eliminating the use of the `CURRENT_TIME` macro in filesystems in favor of `current_fs_time()`, which references the filesystem superblock so that the proper time granularity and range can be enforced. Beyond that, the approaches diverge. 

Rather than go through an intermediate inode timestamp type, so that filesystems can be converted over time, Chinner would [like](<https://lwn.net/Articles/672602/>) to turn that on its head a bit. Start by ensuring that all filesystems that use `timespec` internally call a (for now empty) conversion function to change them to and from the VFS representation. That would eliminate all of the macro changes that were needed when using `inode_timespec`: 

This works, and is much cleaner than propagating the macro nastiness everywhere. IMO vfs_time_to_timespec()/timespec_to_vfs_time would be better named as it describes the conversion exactly. I don't think this is a huge patch, though - it's mainly the setattr/kstat operations that need changing here. 

Internally, time handling in those filesystems could remain unchanged; it would just be a change at the boundary between the filesystem and the VFS. That would isolate the changes that need to be done for the VFS from those that need to be done for the filesystems. Chinner said that all filesystems will need an audit to determine what they need to support post-2038 timestamps, so this decoupling is useful: 

All filesystems will, at least, need auditing. A large number of them will need changes, no matter how we "abstract" the VFS type change, even if it is just for 32->64 bit sign extension bugs. 

Filesystems that have intermediate timestamp formats such as Lustre, NFS, CIFS, etc will need conversion at the vfs/filesystem entry points, and their internals will remain unchanged. Fixing the internals is outside the scope fo the VFS change - the 64 bit VFS inode support stops at the VFS inode/filesystem boundary. 

But Dinamani and Bergmann are leery of an enormous patch set that touches lots of code all over the place. It is both ""ugly and fragile"" as Bergmann [put it](<https://lwn.net/Articles/672610/>), though he suggested at least investigating that path. Both he and Dinamani have made various attempts to find the right approach and they have both run into various walls. Chinner's [suggestion](<https://lwn.net/Articles/672612/>) of how to handle a particular case for the FAT filesystem is not workable, they said. Bergmann [elaborated](<https://lwn.net/Articles/672613/>): 

That puts us back at the 'one big patch' problem: We can't change fat_time_fat2unix() to pass a timespec64 until we also change struct inode. The change may be small, but I see roughly 30 file systems that assign i_?time into or from a local variable or pass it into by reference into a function that is not from VFS. 

So there seems to be an impasse at this point. Dinamani [said](<https://lwn.net/Articles/672615/>) that she would try to convert an example filesystem using the two different methods for comparison purposes. Hopefully that will help point the way toward a solution that leads to as little disruption as possible. A change of this sort is always going to lead to some upheaval, but finding a way to reduce it as much as possible will be good. So far, Dinamani and Bergmann haven't quite found the right approach—or haven't yet convinced Chinner—but it is good to see that kernel developers are thinking about this.
