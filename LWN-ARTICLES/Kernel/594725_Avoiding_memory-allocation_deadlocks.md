---
title: "Avoiding memory-allocation deadlocks"
url: https://lwn.net/Articles/594725/
date: "April 16, 2014"
category: Memory management; gfp t
author: "April 16, 2014 This article was contributed by Neil Brown"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

April 16, 2014

This article was contributed by Neil Brown

There is a saying that you need to spend money to make money, though this apparent paradox is easily resolved with a start-up loan and the discipline of balancing expenses against income. A similar logic applies to the management of memory in an operating system kernel such as Linux: sometimes you need to allocate memory to free memory. Here, too, discipline is needed, though the typical consequences of not being sufficiently careful is not bankruptcy but rather a deadlock. 

The history of how the Linux kernel developed its balance between saving and spending is interesting as a microcosm of how Linux development proceeds, and useful in understanding how to handle future deadlocks when they occur. A good place to start this history is in early 1998 with the introduction of `__GFP_IO` in Linux 2.1.80pre3. 

#### `__GFP_IO` in 2.1.80pre3

Any memory allocation request in Linux includes a `gfp_t` argument, which is a set of flags to guide how the `get_free_page()` function can go about locating a free page. [2.1.80pre3](<https://git.kernel.org/cgit/linux/kernel/git/history/history.git/commit/?id=472bbf0af5bc9f7c933a5d3212b0d765176e728a>) marks a change in this argument's type; it went from being a simple enumerated type to being a bitmask. The concepts embodied in each flag were present previously, but this is the first time that they could be explicitly identified. 

`__GFP_IO` was one of the new flags. If it was set, then `get_free_pages()` was allowed to call `shm_swap()` to write some pages out to swap. If `shm_swap()` needed to allocate any `buffer_head` structures to complete the writeout, it would be careful not to set `__GFP_IO`. Without this protection, an infinite recursion could easily happen, which would quickly exhaust the stack and cause the kernel to crash. 

We have `__GFP_IO` in the kernel today, but, despite having the same name, it is a different flag. Having been introduced for 2.1.80, the original `__GFP_IO` was removed in 2.1.116, to be replaced with... 

#### `PF_MEMALLOC` in 2.1.116

In the [distant past](<https://git.kernel.org/cgit/linux/kernel/git/history/history.git/commit/?id=ac995a26c87ac75983960cbe4085a77b6bbe3e4d>) (August 1998), we did not have change logs of nearly the quality that we have today, so an operating-system archaeologist is left to guess at the reasons for changes. All we can be really sure of is that the (per-request) `__GFP_IO` flag to `get_free_page()` disappeared, and a new per-process flag called `PF_MEMALLOC` appeared to take over the task of avoiding recursion. One clear benefit of this change is that it is more focused in addressing one particular issue: recursion is clearly a per-process issue and so a per-process flag is fitting. Previously, many memory allocation sites would avoid `__GFP_IO` when they didn't really need to, just in case. Now each call site doesn't need to worry about the problem of recursion; that concern is addressed precisely where it is needed. 

The code comments here highlight an important aspect of memory allocation: 

```
* The "PF_MEMALLOC" flag protects us against recursion:
    	 * if we need more memory as part of a swap-out effort we
    	 * will just silently return "success" to tell the page
    	 * allocator to accept the allocation.
```

When possible, `get_free_page()` will just pluck a page off the free list and return it as quickly as it can. When that is not possible, it does not satisfy itself with freeing just one page, but will try to free quite a few, to save work next time. Thus, it is re-stocking that startup loan. A particular consequence of `PF_MEMALLOC` is that the memory allocator won't try too hard to gets lots of pages; it will make do with what it has. 

This means that processes with the `PF_MEMALLOC` flag set will have access to the last dregs of free memory, while other processes will need to go out and free up lots of memory first before they can use any. This property of `PF_MEMALLOC` is still present and somewhat more formal in the current kernel. The memory allocator has a concept of "watermarks" such that, if the amount of free memory is below the chosen watermark, the allocator will try to free more memory rather than return what it has. Different `__GFP` flags can select different watermark levels (min, low, high). `PF_MEMALLOC` causes all watermarks to be ignored; if any memory is available at all, it will be returned. 

`PF_MEMALLOC` effectively says "It is time to stop saving and start spending, or we'll have no product to sell". In consequence of this, `PF_MEMALLOC` is now used more broadly than just for avoiding recursion (though it still has that role). Several kernel threads, such as those for `nbd`, the network block device, `iscsi_tcp`, and the MMC card controller, all set `PF_MEMALLOC`, presumably so they can be sure to get memory whenever they are called upon to write out a page of memory (so it can be freed). 

In contrast, the MTD driver (which manages NAND flash and has a similar role to the MMC card driver) [stopped using](<http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=d5ba1c8ffd0b7d38e1fac48184637e6d6c1197d3>) the `PF_MEMALLOC` flag in 2.6.33 with a comment suggesting it was an inappropriate usage. Whether the other uses in the kernel are still justified is a question too deep for our present discussion. 

#### `__GFP_IO` in 2.2.0pre6

When `__GFP_IO` [reappears](<https://git.kernel.org/cgit/linux/kernel/git/history/history.git/commit/?id=70c27ee94003b5e3741c5d36f5a84ac6cc81ae82>) it has a similar purpose as the original, but for an importantly different reason. To understand that reason, it suffices to look at a comment in the code: 

```
/*
    	 * Don't go down into the swap-out stuff if
    	 * we cannot do I/O! Avoid recursing on FS
    	 * locks etc.
    	 */
```

The concern here still involves recursion, but it also involves locks, such as the per-inode mutex, the page lock, or various others. Calling into a filesystem to write out a page may require taking a lock. If any such lock is held when allocating memory then it is important to avoid calling into any filesystem code that might try to acquire the same lock. In those cases, the code must be careful not to pass `__GFP_IO`; in other cases, it is perfectly safe to include that flag. 

So while `PF_MEMALLOC` avoids the specific recursion of `get_free_page()` calling into `get_free_page()`, `__GFP_IO` is more general and prevents any function holding a lock from calling, through `get_free_page()`, into any other function which might want that lock. The risk here isn't exhausting the stack as with `PF_MEMALLOC`; the risk is a deadlock. 

One might wonder why a `GFP` flag was used for this rather than a process flag, which would effectively say "I am holding a filesystem lock", given that the previous experience with `__GFP_IO` wasn't a success. Like many software designs, it probably just "seemed like a good idea at the time". 

#### `__GFP_FS` in 2.4.5.8

This flag started life named `__GFP_BUFFER` in 2.4.5.1, but didn't really work properly until 2.4.5.8 when it was renamed to `__GFP_FS`. Apparently there was a [thinko](<http://git.kernel.org/cgit/linux/kernel/git/tglx/history.git/commit/?id=75b566af5cc6f64f9ab5b66608ff8ce18098a2b4>) in the original design, which required not only a range of code changes, but also a new name. 

`__GFP_FS` effectively split some functionality away from `__GFP_IO` so that where there was one flag, now there were two. Only three combinations of the two were expected: neither, both, or the new possibility of just the `__GFP_IO` flag being set. This would allow buffers that were already prepared to be written out, but would prohibit any calls into filesystems to prepare those buffers. I/O activity would be permitted, but filesystem activity would not. 

Presumably, the fact that `__GFP_IO` previously had such a broad effect was harming performance, in that it had to be excluded in places where some I/O was still possible. Refining the rules by adding a new flag led to more flexibility, and so fewer impediments to performance. 

#### `PF_FSTRANS` in 2.5.36

This new process flag appeared when [XFS support](<https://git.kernel.org/cgit/linux/kernel/git/tglx/history.git/commit/?id=ef5cc2fd95520561f7e3c8c49b809000dee033ba>) was merged into Linux in late 2002. Its purpose was to indicate that a filesystem transaction (hence the name) was being prepared, meaning that any write to the filesystem would likely block until the transaction processing was complete. The effect of this flag was to exclude `__GFP_FS` from any memory allocation request which happened while `PF_FSTRANS` was set, or at least any request from within the XFS code. Other requests would not be affected, but then other code that allocated memory would be unlikely to be called while the flag was set. 

Another way to see this flag is that, in the same way that the original `__GFP_IO` was converted to `PF_MEMALLOC`, now `__GFP_FS` is being converted to a process flag, too. In this case, the conversion is not complete, though. 

Back in the halcyon days of 2.1.116, removing a flag like `__GFP_IO` was quite straightforward — there were few users and the implications of the change could be easily understood. In the more complex times of 2.5.36, such a step would be far from easy. Carefully adding new functionality is one thing, removing something that is entrenched is quite another, as we have seen with the [Big Kernel Lock](<http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=4ba8216cd90560bc402f52076f64d8546e8aefcb>) and the [`sleep_on()` interface](<https://lwn.net/Articles/593670/>). Allowing either the new flag or the absence of the old to have the same effect is not a big cost and it was best to leave things that were working alone. 

Skipping ahead of ourselves a little to [3.6-rc1](<http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=5cf02d09b50b1e>), the `PF_FSTRANS` flag also gets used by NFS. Rather than setting it during a transaction, NFS sets it while constructing and transmitting an RPC request onto the network, so the name is now slightly less appropriate. Also the effect of the flag on NFS is not exactly to clear `__GFP_FS`, but simply to avoid a call to transmit a `COMMIT` request inside `nfs_release_page()`, which is also avoided if `__GFP_FS` is missing. This is a superficially different usage than the usage by XFS, but it has a generally similar effect for a generally similar reason. Modifying the flag to have a more global effect of clearing `GFP_FS` and maybe renaming it to `PF_MEMALLOC_NOFS` might not be a bad idea. 

#### `set_gfp_allowed_mask()` in 2.6.34

This function actually [appeared](<http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=7e85ee0c1d15ca5f8bff0f514f158eba1742dd87>) in [2.6.31](<http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=dcce284a259373f9e5570f2e33f79eca84fcf565>), but becomes more interesting in 2.6.34. 

`gfp_allowed_mask` is a global variable holding a set of `GFP` flags which are allowed to be honored — all others are ignored. In particular, `__GFP_FS`, `__GFP_IO`, and `__GFP_WAIT` (which generally allows `get_free_page()` to wait for memory to be freed by other processes) are sometimes disabled via this mechanism. Thus it is a bit like `PF_FSTRANS`, except that it affects more processes and disables more flags. 

`gfp_allowed_mask` came about while providing support for `kmalloc()` earlier in the boot process. During early boot, interrupts are still disabled and any attempt to allocate memory with `__GFP_WAIT` or certain other flags can trigger a warning from the `lockdep` checker. It would be surprising if memory were so tight during boot that the allocator actually needed to wait, but getting rid of warnings is generally a good thing, so `gfp_allowed_mask` was initialized to exclude the three flags mentioned, and these were added back in once the boot process was complete. 

One thing we have learned over the years is that boot isn't as special as we sometimes think: whether it is suspend and resume, or hotplug hardware which teaches us this, it seems to be a lesson we keep finding new perspectives on. In that light, it is perhaps unsurprising that, in [2.6.34](<http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=452aa6999e6703ffbddd7f6ea124d3968915f3e3>), the use of this mask was extended to cover suspend and resume (though an [early version](<https://lkml.org/lkml/2009/6/12/82>) of the original patch did mention the importance of suspend). 

In the case of memory allocation deadlocks, the suspend case is more significant than the boot case. During boot there is usually lots of free memory — not so during suspend, when we may well be short of memory. It wasn't warnings that prompted this change, but genuine deadlocks. 

Suspend and resume are largely orderly processes, with devices being put to sleep in sequence, and then woken again in the reverse sequence. So it would not be enough just for block devices to avoid using `__GFP_IO` (which they already do). Rather, every driver must avoid the `__GFP_IO` flag, and others, as the target block device of some write request, might be sequenced with this driver so that it is already asleep, and will not awake before this one is completely awake. 

Having a system-wide setting to disable these flags may be a bit excessive — just the process which is sequencing suspend might be sufficient — but it is certainly an easy fix and, as it cannot affect normal running of the system, it is thus a safe fix. 

#### `PF_MEMALLOC_NOIO` in 3.9-rc1

Just as suspend/resume has taught us that boot-time is not that much of a special case, so too runtime power management has taught us that suspend isn't all that much of a special case either. If a block device is runtime-suspended to save power, then obviously it cannot handle requests to write out a dirty page of memory until it has woken up, and until any devices it depends on (a USB controller, a PCI bus) are awake too. So none of these devices can safely perform memory allocation using `__GFP_IO`. 

In order to ensure this, we could use `set_gfp_allowed_mask()` while a device was suspending or resuming, but if multiple such devices were suspending or resuming we could easily lose track of when to restore the right mask. So [this change](<http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=21caf2fc1931b485483ddd254b634fa8f0099963>) introduces a process flag much like `PF_FSTRANS`, only to disable `__GFP_IO` rather than `__GFP_FS`. It also takes care to record the old value whenever the flag is set, and restore that old value when done. To know when to set this flag, a `memalloc_noio` flag is introduced for each device; it is then propagated into the parents in the device tree. `PF_MEMALLOC_NOIO` is set whenever calling into the power management code for any device with `memalloc_noio` set. 

As both the early boot processing and the suspend/resume processing are largely single-threaded (or have dedicated threads), it is quite possible that setting `PF_MEMALLOC_NOIO` and `PF_FSTRANS` on those threads would be a sufficient alternative to using `set_gfp_allowed_mask()`. However, as there is no clear benefit from such a change, and no clear certainty that it would work, it is safer, once again, to leave that which works alone. 

#### Patterns that emerge

Amid all these details there are a couple of patterns which stand out. 

The first is repeated refinement of the "avoid recursion" concept. At first it was implicit in an enumerated value passed to `get_free_page()`, then it was made explicit in the first `__GFP_IO`, and then the `PF_MEMALLOC` flag. Next it was extended to cover more subtle forms of recursion with the second version of `__GFP_IO` and, finally, that was split into two separate flags to express an even wider range of recursion scenarios that can be separately avoided. 

It is conceivable that there is room for further refinement. We could have separate flags for different sorts of locks -- one for page locks and one for inode locks, for example. There is no evidence that this would presently be useful, but Linux isn't really finished yet, so we just don't know. 

The second pattern is the repeated discovery that just having a `GFP` flag often isn't enough — three times a new process flag was added because sometimes it isn't just a single allocation that needs to be controlled, but all allocations made by a given process. Is it only a matter of time before we get either a process flag which disables `__GFP_WAIT` or a per-process `gfp_allowed_mask`? 

As a footnote to this pattern, it is amusing that in [3.6-rc1](<http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=b37f1dd0f543d9714f96c2f9b9f74f7bdfdfdf31>), as part of adding support for swap-over-NFS, a new flag, `__GFP_MEMALLOC`, was added which has much the same effect as `PF_MEMALLOC` in ignoring the normal low-watermarks and providing access to the last reserves of memory. This, together with the per-socket `sk_allocation` mask, allows certain TCP sockets (those which NFS is performing swap over) to access those last reserves to make sure that swap-out always succeeds. Clearly there is need for both `GFP` flags and process flags, as well as some per-device and per-socket flags. 

#### We've not seen the last of this

While studying history can be generally enlightening, it can also be specifically useful as it is in this case. Next week, we will use this understanding of memory allocation deadlocks to explore some deadlocks which have long been possible in a certain configuration, but which now need to be removed.
