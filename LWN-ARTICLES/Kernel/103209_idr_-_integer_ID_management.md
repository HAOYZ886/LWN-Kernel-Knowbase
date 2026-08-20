---
title: "idr - integer ID management"
url: https://lwn.net/Articles/103209/
date: "September 22, 2004"
category: idr
author: 
---

There has been a fair number of patches in recent times which convert one part or other of the kernel over to the "idr" facility. Idr is a set of library functions for the management of small integer ID numbers. In essence, an idr object can be thought of as a sparse array mapping integer IDs onto arbitrary pointers, with a "get me an available entry" function as well. This code was first added in February, 2003 as part of the POSIX clocks patch, and has seen various tweaks since. 

Working with idr requires including `<linux/idr.h>`. Creating a new idr object is simply a matter of allocating a `struct idr` and passing it to: 

```
void idr_init(struct idr *idp);
```

The interface for allocating new IDs is somewhat unintuitive and interesting. The authors decided to separate out the parts of the ID allocation process which may require getting memory from the system; the idea was that the memory allocation could be done with no locks held, while the actual generation of an ID number could be done in a locked state. Thus, before allocating a new ID, one must call: 

```
int idr_pre_get(struct idr *idp, unsigned int gfp_mask);
```

This function will get set up to allocate a new ID number, allocating memory (with the given `gfp_mask`) if necessary. Contrary to the usual conventions, the return value will be zero if something goes wrong, nonzero otherwise. 

Once that is done, a new ID can be allocated with either of: 

```
int idr_get_new(struct idr *idp, void *ptr, int *id);
        int idr_get_new_above(struct idr *idp, void *ptr, int start_id, int *id);
```

The first form gets the next available ID number, stores it in `id`, and associates it with the given `ptr` internally. If you wish to specify a minimum value for the new ID, use `idr_get_new_above()` instead. If all goes well, the return value will be zero; if no more IDs can be allocated, `-ENOSPC` will be returned. 

Imagine a situation where two processors are both looking to allocate a new ID. Both call `idr_pre_get()`, guaranteeing that enough memory exists to allocate at least one more ID. Then one processor swoops in and grabs that ID, leaving no memory for the other. In that case, `idr_get_new()` will not attempt to allocate more memory; it will, instead, return `-EAGAIN`. At that point, the code should emit a heavy sigh, release its locks, and go back to the `idr_pre_get()` stage. Thus, ID allocation code can look something like this: 

```
again:
    	if (idr_pre_get(&my_idr, GFP_KERNEL) == 0) {
    		/* No memory, give up entirely */
    	}
    	spin_lock(&my_lock);
    	result = idr_get_new(&my_idr, &target, &id);
    	if (result == -EAGAIN) {
    		sigh();
    		spin_unlock(&my_lock);
    		goto again;
    	}
```

It should be noted that calls to `idr_get_new()` (and most other idr functions) must be serialized by some sort of lock, or unpleasant things could happen. `idr_pre_get()` can sleep, however, and should not be called under lock. 

Looking up an existing ID is much simpler: 

```
void *idr_find(struct idr *idp, int id);
```

The return value will be the pointer associated with the given `id`, or `NULL` otherwise. 

To deallocate an ID, use: 

```
void idr_remove(struct idr *idp, int id);
```

With these functions, kernel code can generate ID numbers to use as minor device numbers, inode numbers, or in any other place where small integer IDs are useful. 

There is one more interesting twist to the idr code: it does (almost) nothing to help users detect reused ID numbers. When an object is destroyed, it may not be possible to tell whether anybody still has its ID number around or not. When some part of the kernel comes along with an ID number, it would be nice to know that refers to a currently-existing object, rather than being left over from some previous time. 

The idr code makes it possible for callers to perform this check by ignoring the high-order bits in the ID number. Here, "high-order" is defined as "all the bits which are not needed to represent the largest allocated ID." By putting some sort of unique information in the upper part of the ID (and by limiting the maximum ID number which can be used), idr users can turn the small ID numbers into unique identifiers. The POSIX timer and SCTP code use idr in this way; most of the other in-kernel users treat idr as a sort of unique number generation service and do not perform this sort of check.
