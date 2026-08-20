---
title: The ongoing fallocate() story
url: https://lwn.net/Articles/240571/
date: "July 3, 2007"
category: fallocate
author: 
---

The proposed `fallocate()` system call, which exists to allow an application to preallocate blocks for a file, was [covered here](<http://lwn.net/Articles/226710/>) back in March. Since then there has been quite a bit of discussion, but there is still no `fallocate()` system call in the mainline - and it's not clear that there will be in 2.6.23 either. There is [a new version of the `fallocate()` patch](<http://lwn.net/Articles/239738/>) in circulation, so it seems like a good time to catch up with what is going on. 

Back in March, the proposed interface was: 

```
long fallocate(int fd, int mode, loff_t offset, loff_t len);
```

It turns out that this specific arrangement of parameters is hard to support on some architectures - the S/390 architecture in particular. Various alternatives were proposed, but getting something that everybody liked proved difficult. In the end, the above prototype is still being used. The S/390 architecture code will have to do some extra bit shuffling to be able to implement this call, but that apparently is the best way to go. 

That does not mean that the interface discussions are done, though. The current version of the patch now has four possibilities for `mode`: 

  * `FA_ALLOCATE` will allocate the requested space at the given `offset`. If this call makes the file longer, the reported size of the file will be increased accordingly, making the allocated blocks part of the file immediately. 

  * `FA_RESV_SPACE` preallocates blocks, but does not change the size of the file. So the newly allocated blocks, if past the end of the file, will not appear to be present until the application writes to them (or increases the size of the file in some other way). 

  * `FA_DEALLOCATE` returns previously-allocated blocks to the system. The size of the file will be changed if the deallocated blocks are at the end. 

  * `FA_UNRESV_SPACE` returns the blocks to the system, but does not change the size of the file. 

As an example of how the last two operations differ, consider what happens if an application uses `fallocate()` to remove the last block from a file. If that block was removed with `FA_DEALLOCATE`, a subsequent attempt to read that block will return no data - the offset where that block was is now past the end of the file. If, instead, the block is removed with `FA_UNRESV_SPACE`, an attempt to read it will return a block full of zeros. 

It turns out that there are some differing opinions on how this interface should work. A trivial change which has been requested is that the `FA_` prefix be changed to `FALLOC_` \- this change is likely to be made. But it seems there's a number of other flags that people would like to see: 

  * `FALLOC_ZERO_SPACE` would write zeros to the requested range - even if that range is already allocated to the file. This feature would be useful because some filesystems can quickly mark the affected range as being uninitialized rather than actually writing zeros to all of those blocks. 

  * `FALLOC_MKSWAP` would allocate the space, mark it initialized, but not actually zero out the blocks. The newly-allocated blocks would thus still contain whatever data the previous user left there. This operation, which would clearly have to be privileged, is intended to make it possible to create a swap file in a very quick way. It would require very little in the way of in-kernel memory allocations to implement, making it a useful way to add an emergency swap file to a system which has gone into an out-of-memory condition. 

  * `FALLOC_FL_ERR_FREE` would be an additional flag which would affect error handling; in particular, it would control behavior when the filesystem runs out of space part way through an allocation request. If this flag is set, the blocks which were successfully preallocated would be freed; otherwise they would be left in place. There is some opposition to this flag; it may be left out in favor of an official "all or nothing" policy for preallocations. 

  * `FALLOC_FL_NO_MTIME` and `FALLOC_FL_NO_CTIME` would prevent the filesystem from updating the modification times associated with the file. 

All told, it's a significant number of new features - enough that some people are starting to wonder if `fallocate()` is the right approach after all. Christoph Hellwig, in particular, has started to complain; he suggests adding something small which would be able to implement `posix_fallocate()` and no more. Block deletion, he says, is a different function and should be done with a different system call, and the other features need more thought (and aggressive weeding). So it's unclear where this patch set will go and whether it will be considered ready for 2.6.23.
