---
title: "Who needs /dev/kmem?"
url: https://lwn.net/Articles/147901/
date: "August 16, 2005"
category: devkmem
author: 
---

Steven Rostedt recently [ran into a little problem](<https://lwn.net/Articles/147902/>). He was trying to read the value of a kernel variable using `/dev/kmem`, but his attempts returned an I/O error. The resulting inquiry has led to people asking whether `/dev/kmem` should exist at all. 

Unix-like systems have, since nearly the beginning, offered a couple of character device files called `/dev/mem` and `/dev/kmem`. `/dev/mem` is a straightforward window into main memory; a suitably privileged application can access any physical page in the system by opening `/dev/mem` and seeking to its physical address. This special file can also be used to map parts of the physical address space directly into a process's virtual space, though this only works for addresses which do not correspond to RAM (the X server uses it, for example, to access the video adapter's memory and control registers). 

`/dev/kmem` is supposed to be different in that its window is from the kernel's point of view. A valid offset in `/dev/kmem` would be a kernel virtual address - these addresses look much like physical addresses, but they are not. On commonly-configured i386 systems, for example, the base of the kernel's virtual address space is at `0xc0000000`. The code which implements `mmap()` for `/dev/kmem` looks like this in 2.6.12: 

```
if (!pfn_valid(vma->vm_pgoff))
    		return -EIO;
    	val = (u64)vma->vm_pgoff << PAGE_SHIFT;
    	vma->vm_pgoff = __pa(val) >> PAGE_SHIFT;
    	return mmap_mem(file, vma);
```

The idea is to turn the kernel virtual address into a physical address (using `__pa()`), then use the regular `/dev/mem` mapping function. The problem, of course, is that the `pfn_valid()` test is performed before the given page frame number has been moved into the physical space; thus, any attempt to map an address in the kernel's virtual space will return `-EIO` \- except on some systems with large amounts of physical memory, and, even then, the result will not be what the programmer was after. This mistake would almost certainly be a security hole, except that only root can access `/dev/kmem` in the first place. 

Linus has merged [a simple fix](<https://lwn.net/Articles/147903/>) for 2.6.13. It does not even try to solve the whole problem, in that it still fails to properly check the full address range requested by the application. But the real question that has come out of this episode is: is there any reason to keep `/dev/kmem` around? The fact that it has been broken for some time suggests that there are not a whole lot of users out there. It has been suggested that root kits are the largest user community for this kind of access, but there are no forward compatibility guarantees for root kit authors. The Fedora kernel, as it turns out, has not supported `/dev/kmem` for a long time. 

Removing a feature like that is not in the cards for 2.6.13. But, unless some sort of important user shows up, chances are that `/dev/kmem` will not survive into 2.6.14. Anybody who would be inconvenienced by that change should speak up soon.
