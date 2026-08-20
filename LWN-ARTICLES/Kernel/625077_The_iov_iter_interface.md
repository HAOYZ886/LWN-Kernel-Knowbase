---
title: The iov_iter interface
url: https://lwn.net/Articles/625077/
date: "December 9, 2014"
category: iov iter
author: "By Jonathan Corbet December 9, 2014"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
December 9, 2014

One of the most common tasks in the kernel is processing a buffer of data supplied by user space, possibly in several chunks. Perhaps unsurprisingly, this is a task that kernel code often gets wrong, leading to bugs and, possibly, security problems. The kernel contains a primitive (called "`iov_iter`") meant to make this task simpler. While `iov_iter` use is mostly confined to the memory-management and filesystem layers currently, it is slowly spreading out into other parts of the kernel. This interface is currently undocumented, a situation this article will attempt to remedy. 

The `iov_iter` concept is not new; it was first [added by Nick Piggin](<http://git.kernel.org/linus/2f718ffc16c43a435d12919c75dbfad518abd056>) for the 2.6.24 kernel in 2007. But there has been an effort over the last year to expand this API and use it in more parts of the kernel; the 3.19 merge window should see it making its first steps into the networking subsystem, for example. 

An `iov_iter` structure is essentially an iterator for working through an `iovec` structure, defined in `<uapi/linux/uio.h>` as: 

```
struct iovec
        {
    	void __user *iov_base;
    	__kernel_size_t iov_len;
        };
```

This structure matches the user-space `iovec` structure defined by POSIX and used with system calls like `readv()`. As the "vec" portion of the name would suggest, `iovec` structures tend to come in arrays; as a whole, an `iovec` describes a buffer that may be scattered in both physical and virtual memory. 

The actual `iov_iter` structure is defined in `<linux/uio.h>`: 

```
struct iov_iter {
    	int type;
    	size_t iov_offset;
    	size_t count;
    	const struct iovec *iov; /* SIMPLIFIED - see below */
    	unsigned long nr_segs;
        };
```

The `type` field describes the type of the iterator. It is a bitmask containing, among other things, either `READ` or `WRITE` depending on whether data is being read into the iterator or written from it. The data direction, thus, refers not to the iterator itself, but to the other part of the data transaction; an `iov_iter` created with a `type` of `READ` will be written to. 

Beyond that, `iov_offset` contains the offset to the first byte of interesting data in the first `iovec` pointed to by `iov`. The total amount of data pointed to by the `iovec` array is stored in `count`, while the number of `iovec` structures is stored in `nr_segs`. Note that most of these fields will change as code "iterates" through the buffer. They describe a cursor into the buffer, rather than the buffer as a whole. 

#### Working with struct iov_iter

Before use, an `iov_iter` must be initialized to contain an (already populated) `iovec` with: 

```
void iov_iter_init(struct iov_iter *i, int direction,
    		       const struct iovec *iov, unsigned long nr_segs,
    		       size_t count);
```

Then, for example, data can be moved between the iterator and user space with either of: 

```
size_t copy_to_iter(void *addr, size_t bytes, struct iov_iter *i);
        size_t copy_from_iter(void *addr, size_t bytes, struct iov_iter *i);
```

The naming here can be a little confusing until one gets the hang of it. A call to `copy_to_iter()` will copy `bytes` data _from_ the buffer at `addr` to the user-space buffer indicated by the iterator. So `copy_to_iter()` can be thought of as being like a variant of `copy_to_user()` that takes an iterator rather than a single buffer. Similarly, `copy_from_iter()` will copy the data from the user-space buffer to `addr`. The similarity with `copy_to_user()` continues through to the return value, which is the number of bytes _not_ copied. 

Note that these calls will "advance" the iterator through the buffer to correspond to the amount of data transferred. In other words, the `iov_offset`, `count`, `nr_segs`, and `iov` fields of the iterator will all be changed as needed. So two calls to `copy_from_iter()` will copy two successive areas from user space. Among other things, this means that the code owning the iterator must remember the base address for the `iovec` array, since the `iov` value in the `iov_iter` structure may change. 

Various other functions exist. To move data referenced by a `page` structure into or out of an iterator, use: 

```
size_t copy_page_to_iter(struct page *page, size_t offset, size_t bytes,
    			     struct iov_iter *i);
        size_t copy_page_from_iter(struct page *page, size_t offset, size_t bytes,
    			       struct iov_iter *i);
```

Only the single `page` provided will be copied to or from, so these functions should not be asked to copy data that would cross the page boundary. 

Code running in atomic context can attempt to obtain data from user space with: 

```
size_t iov_iter_copy_from_user_atomic(struct page *page, struct iov_iter *i,
    					  unsigned long offset, size_t bytes);
```

Since this copy will be done in atomic mode, it will only succeed if the data is already resident in RAM; callers must thus be prepared for a higher-than-normal chance of failure. 

If it is necessary to map the user-space buffer into the kernel, one of these calls can be used: 

```
ssize_t iov_iter_get_pages(struct iov_iter *i, struct page **pages,
                                   size_t maxsize, unsigned maxpages, size_t *start);
        ssize_t iov_iter_get_pages_alloc(struct iov_iter *i, struct page ***pages, 
        	    			     size_t maxsize, size_t *start);
```

Either function turns into a call to `get_user_pages_fast()`, causing (hopefully) the pages to be brought in and their locations stored in the `pages` array. The difference between them is that `iov_iter_get_pages()` expects the `pages` array to be allocated by the caller, while `iov_iter_get_pages_alloc()` will do the allocation itself. In that case, the array returned in `pages` must eventually be freed with `kvfree()`, since it might have been allocated with either `kmalloc()` or `vmalloc()`. 

Advancing through the iterator without moving any data can be done with: 

```
void iov_iter_advance(struct iov_iter *i, size_t size);
```

The buffer referred to by an iterator (or a portion thereof) can be cleared with: 

```
size_t iov_iter_zero(size_t bytes, struct iov_iter *i);
```

Information about the iterator is available from a number of helper functions: 

```
size_t iov_iter_single_seg_count(const struct iov_iter *i);
        int iov_iter_npages(const struct iov_iter *i, int maxpages);
        size_t iov_length(const struct iovec *iov, unsigned long nr_segs);
```

A call to `iov_iter_single_seg_count()` returns the length of the data in the first segment of the buffer. `iov_iter_npages()` reports the number of pages occupied by the buffer in the iterator, while `iov_length()` returns the total data length. The latter function must be used with care, since it trusts the `len` field in the `iovec` structures. If that data comes from user space, it could cause integer overflows in the kernel. 

#### Not just iovecs

The definition of `struct iov_iter` shown above does not quite match what is actually found in the kernel. Instead of a single field for the `iov` array, the real structure has (in 3.18): 

```
union {
    	const struct iovec *iov;
    	const struct bio_vec *bvec;
        };
```

In other words, the `iov_iter` structure is also set up to work with the BIO structures used by the block layer. Such iterators are marked by having `ITER_BVEC` include in the `type` field bitmask. Once such an iterator is created, all of the above calls will work with it as if it were an "ordinary" iterator using `iovec` structures. Currently, the use of BIO-based iterators in the kernel is minimal; they can only be found in the swap and `splice()` code. 

#### Coming in 3.19

The 3.19 kernel is likely to see a substantial rewrite of the `iov_iter` code aimed at reducing the vast amount of boilerplate code needed to implement all of the above-mentioned functions. The code is indeed shorter afterward, but at the cost of introducing a fair amount of mildly frightening preprocessor macro magic to generate the needed boilerplate on demand. 

The `iov_iter` code already works if the "user-space" buffer is actually located in kernel space. In 3.19, things will be formalized and optimized a bit. Such an iterator will be created with: 

```
void iov_iter_kvec(struct iov_iter *i, int direction,
    		       const struct kvec *iov, unsigned long nr_segs,
    		       size_t count);
```

There will also be a new `kvec` field added to the union shown above for this case. 

Finally, some functions have been added to help with the networking case; it will be possible, for example, to copy a buffer and generate a checksum in the process. 

The end result is that the `iov_iter` interface is slowly becoming the standard way of hiding many of the complexities associated with handling user-space buffers. We can expect to see its use encouraged in more places in the future. It only took seven years or so, but `iov_iter` appears to be reaching a point of being an interface that most kernel developers will want to be aware of.
