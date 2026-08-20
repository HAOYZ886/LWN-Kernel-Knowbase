---
title: What became of getrandom() in the vDSO
url: https://lwn.net/Articles/983186/
date: "July 25, 2024"
category: "Random numbers; Security-Random number generation"
author: "By Jonathan Corbet July 25, 2024"
---

By **Jonathan Corbet**  
July 25, 2024

In [the previous episode](<https://lwn.net/Articles/980447/>) of the `vgetrandom()` story, Jason Donenfeld had put together a version of the [`getrandom()`](<https://man7.org/linux/man-pages/man2/getrandom.2.html>) system call that ran in user space, significantly improving performance for applications that need a lot of random data while retaining all of the guarantees provided by the system call. At that time, it seemed that a consensus had built around the implementation and that it was headed toward the mainline in that form. A few milliseconds after that article was posted, though, a Linus-Torvalds-shaped obstacle appeared in its path. That obstacle has been overcome and this work has now been merged for the 6.11 kernel, but its form has changed somewhat. 

Torvalds initially [rejected](<https://lwn.net/ml/all/CAHk-=wiGk+1eNy4Vk6QsEgM=Ru3jE40qrDwgq_CSKgqwLgMdRg@mail.gmail.com>) the idea of a vDSO implementation entirely, saying that there was no clear use case for it. At most, he [said](<https://lwn.net/ml/all/CAHk-=wh47WSNQYuSWqdu_8XeRzfpWbozzTDL6KtkGbSmLrWU4g@mail.gmail.com>), the kernel should export a generation counter to inform user-space random-number generators that they should reseed themselves; anything beyond that, he said, was more than the kernel needed to provide. After a fair amount of back-and-forth with Donenfeld, who made the point that he did not want to expose the internal functioning of the kernel's random-number generator to user space, Torvalds reluctantly [agreed](<https://lwn.net/ml/all/CAHk-=wjCmw1L42W-o=pW7_i=nJK5r0_HFQTWD_agKWGt4hE7JQ@mail.gmail.com>) to take another look and reconsider. 

When [he came back](<https://lwn.net/ml/all/CAHk-=win2mesMNEfL-KZQ_jk1YH8N8dL9r=7XOLp28_WMazpVg@mail.gmail.com>) a couple of hours later, his biggest complaint centered around the new `vgetrandom_alloc()` system call, which was added to allocate the special memory needed to hold the per-thread state used by `vgetrandom()`. The ability to allocate memory that could be dropped by the kernel if needed had been requested before, he said, and could be useful in other settings. But it had been made available in a specialized form that was only suitable for `vgetrandom()`; that would lead developers to misuse that system call to allocate memory for other purposes. ""And that nightmare has to be avoided"". 

His suggestion was that `vgetrandom_alloc()` should go away, and that the ability to allocate droppable memory should, instead, become just another [`mmap()`](<https://man7.org/linux/man-pages/man2/mmap.2.html>) flag. Donenfeld made that change, with the result that `mmap()` in 6.11 will support the `MAP_DROPPABLE` flag; that flag will be mandatory for memory allocated for use with `vgetrandom()`. Developers will be able to allocate droppable memory for other purposes, if desired, without having to use a special-purpose system call. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Allocating memory was only one of the reasons to call `vgetrandom_alloc()`, though; that call also informed the caller about how much memory was needed to hold the per-thread state. That information is now obtained with a special call to `vgetrandom()`, which retains the same prototype from previous attempts: 

```
ssize_t vgetrandom(void *buffer, size_t len, unsigned int flags,
                           void *opaque_state, size_t opaque_len);
```

If this function is called with the `buffer`, `len`, and `flags` parameters all set to zero and `opaque_len` set to `~0UL` then, rather than generating random data, `vgetrandom()` will fill the memory pointed to by `opaque_state` with this structure: 

```
struct vgetrandom_opaque_params {
    	__u32 size_of_opaque_state;
    	__u32 mmap_prot;
    	__u32 mmap_flags;
    	__u32 reserved[13];
        };
```

The caller can then allocate the needed memory to store the per-thread state for as many threads as might be needed, passing the provided `mmap_prot` and `mmap_flags` values directly to `mmap()`. It is the caller's responsibility to ensure that the allocated memory does not cross a page boundary. 

Once this memory has been allocated, user space can use `vgetrandom()` as described in the previous article — essentially as a drop-in replacement for the `getrandom()` system call. This should happen with no action needed by most developers, since it is expected that the C libraries will handle the allocation of the state memory and calls to the vDSO implementation. Application developers should get the newer, faster `getrandom()` for free with a library upgrade.
