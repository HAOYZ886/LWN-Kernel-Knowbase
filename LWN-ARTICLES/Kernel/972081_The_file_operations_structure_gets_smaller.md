---
title: The file_operations structure gets smaller
url: https://lwn.net/Articles/972081/
date: "May 3, 2024"
category: struct file operations
author: "By Jonathan Corbet May 3, 2024"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
May 3, 2024

Kernel developers are encouraged to send their changes in small batches as a way of making life easier for reviewers. So when a longtime developer and maintainer hits the list with a 437-patch series touching 859 files, eyebrows are certain to head skyward. Specifically, [this series from Jens Axboe](<https://lwn.net/ml/linux-kernel/20240411153126.16201-1-axboe@kernel.dk/>) is cleaning up one of the core abstractions that has been part of the Linux kernel almost since the beginning; authors of device drivers (among others) will have to take note. 

#### The origin of `struct file_operations`

In the beginning, the Linux kernel lacked any sort of virtual filesystem layer. See, for example, the [0.01 implementation](<https://elixir.bootlin.com/linux/0.01/source/fs/read_write.c#L49>) of [`read()`](<https://man7.org/linux/man-pages/man2/read.2.html>), which contained explicit checks for each possible file-descriptor type. That approach worked to get an initial kernel to boot but, before long, Linus Torvalds realized that it would not scale well. As developers sought to add more device types, and to implement more than one filesystem type, the need for an abstraction layer became more urgent. 

The Linux 0.95 release, which came out in March 1992, brought a number of changes, including a switch to the GPL license. It also added the first pieces of what was to become the kernel's virtual filesystem layer. A core piece of that layer was [the first `file_operations` structure](<https://elixir.bootlin.com/linux/0.95/source/include/linux/fs.h#L139>), defined, in its entirety, as: 

```
struct file_operations {
    	int (*lseek) (struct inode *, struct file *, off_t, int);
    	int (*read) (struct inode *, struct file *, char *, int);
    	int (*write) (struct inode *, struct file *, char *, int);
        };
```

This structure contains the pointers to the functions needed to implement specific system calls on anything that can be represented by a file descriptor. Rather than use an extended `if`-`then`-`else` sequence to determine which type of file was being operated on, the kernel could just do an indirect call to the appropriate `file_operations` member. As might be expected, the most fundamental operations — reading, writing, and seeking — showed up here first. In early versions of the kernel, there wasn't much else that one could do with a file descriptor. 

The `file_operations` structure grew from there. The [1.0 version of this structure](<https://elixir.bootlin.com/linux/1.0/source/include/linux/fs.h#L265>) included ten members, implementing system calls like [`readdir()`](<https://man7.org/linux/man-pages/man2/readdir.2.html>), [`ioctl()`](<https://man7.org/linux/man-pages/man2/ioctl.2.html>), and [`mmap()`](<https://man7.org/linux/man-pages/man2/mmap.2.html>). The [2.0 version of `struct file_operations`](<https://elixir.bootlin.com/linux/2.0/source/include/linux/fs.h#L455>) had 13 members, and [2.2](<https://elixir.bootlin.com/linux/2.2.0/source/include/linux/fs.h#L576>) added two more. Through all of this history, the `read()` and `write()` members remained the way to read from and write to a file descriptor, though their prototypes changed somewhat. 

#### The plot thickens

The 2.4 release, made at the beginning of 2001, included [a version of `struct file_operations`](<https://elixir.bootlin.com/linux/2.4.0/source/include/linux/fs.h#L773>) with these new members: 

```
ssize_t (*readv) (struct file *, const struct iovec *, unsigned long, loff_t *);
        ssize_t (*writev) (struct file *, const struct iovec *, unsigned long, loff_t *);
```

User-space developers often needed the ability to perform scatter/gather I/O — operations involving multiple segments of memory that needed to be transferred in a single operation. In response, the kernel gained support for [`readv()` and `writev()`](<https://man7.org/linux/man-pages/man2/readv.2.html>) but, to properly support these system calls, the kernel needed to pass them down to the underlying implementations. The new members, which took an array of [`iovec` structures](<https://elixir.bootlin.com/linux/2.4.0/source/include/linux/uio.h#L16>) containing an address (in user space) and size for each segment, were added for this purpose. For device drivers or filesystems that did not implement the new functions, the kernel would emulate them with a series of `read()` or `write()` calls instead. 

Subsequent work added many more members to `struct file_operations`, including other variants of `read()` and `write()`. `aio_read()` and `aio_write()`, used to implement the kernel's somewhat unloved [asynchronous I/O mechanism](<https://man7.org/linux/man-pages/man2/io_submit.2.html>), went into the 2.5.33 development release. `splice_read()` and `splice_write()`, implementing the [`splice()`](<https://man7.org/linux/man-pages/man2/splice.2.html>) system call, were [added](<https://git.kernel.org/linus/5274f052e7b>) for 2.6.17. Removals of `file_operations` members, like the removal of kernel code in general, was rare, but `readv()` and `writev()` were [removed in 2.6.19](<https://git.kernel.org/linus/ee0b3e671baf>) after all users were switched to use `aio_read()` and `aio_write()` instead. 

The [3.16 version of `struct file_operations`](<https://elixir.bootlin.com/linux/v3.16/source/include/linux/fs.h#L1456>), had grown to 27 members, including these additions indicating a new approach to I/O within the kernel: 

```
ssize_t (*read_iter) (struct kiocb *, struct iov_iter *);
        ssize_t (*write_iter) (struct kiocb *, struct iov_iter *);
```

Increasingly, I/O operations were being initiated from the kernel, not just from from user space; they often involved multiple segments and needed to be executed asynchronously. The data buffers involved could be referenced in a number of ways. The `iov_iter` structure used to describe these more complex I/O operations [looked like this](<https://elixir.bootlin.com/linux/v3.16/source/include/linux/uio.h#L22>) at the time: 

```
struct iov_iter {
    	int type;
    	size_t iov_offset;
    	size_t count;
    	union {
    	    const struct iovec *iov;
    	    const struct bio_vec *bvec;
    	};
    	unsigned long nr_segs;
        };
```

The key distinguishing feature of this structure is related to the `type` field. If it was `ITER_IOVEC`, then the `iov` union member contained an array of segments using user-space addresses. If it was, instead, `ITER_KVEC`, then the addresses were in kernel space. And if `type` was `ITER_BVEC`, then the `bvec` field pointed to an array of `bio` structures (used to describe block I/O requests). An I/O API defined in this way could be called from a number of contexts and would work regardless of whether the operation was initiated from user space or from within the kernel. 

The [`kiocb` structure](<https://elixir.bootlin.com/linux/v3.16/source/include/linux/aio.h#L17>) is used by the kernel to coordinate asynchronous I/O operations. Drivers are not required to implement asynchronous I/O (though they may not perform as well if they don't), but if they do implement it, they need the information in this structure. The use of `struct kiocb` reflects the fact that, among other goals, the new methods were intended to replace `aio_read()` and `aio_write()`, which were duly [removed](<http://git.kernel.org/linus/8436318205b9>) for the 4.0 release. 

#### `struct iov_iter` everywhere

Over time, `struct iov_iter` has evolved and become rather more complex; see [the 6.8 version](<https://elixir.bootlin.com/linux/v6.8/source/include/linux/uio.h#L22>) for the details. The kernel has also accumulated a set of helpers that free code from dealing with that complexity much of the time. Meanwhile, [`struct file_operations` in 6.8](<https://elixir.bootlin.com/linux/v6.8/source/include/linux/fs.h#L1983>) is up to 32 callable members. But, through all of this change, `read()` and `write()` have remained essentially unchanged, even though they only handle the simplest of I/O operations in what has become a complicated world. 

Axboe has decided that, perhaps, those two members have reached the end of their useful life: 

> 10 years ago we added ->read_iter() and ->write_iter() to struct file_operations. These are great, as they pass in an iov_iter rather than a user buffer + length, and they also take a struct kiocb rather than just a file. Since then we've had two paths for any read or write - one legacy one that can't do per-IO hints like "This read should be non-blocking", they strictly only work with O_NONBLOCK on the file, and a newer one that supports everything the old path does and a bunch more. 

Since `read_iter()` and `write_iter()` can do everything that `read()` and `write()` can do, it makes sense to simply remove the older members. The only problem is, of course, there is a lot of code that only implements `read()` and `write()` in the kernel; much of it is in drivers that may not have seen significant development (or even use) in years. Some of them surely are being used, though, and breaking them would undoubtedly increase the (already high) level of grumpiness on the net. 

Many modules that use the older interface can, with some effort, be converted to use `read_iter()` and `write_iter()` instead, perhaps gaining functionality in the process. But there are a _lot_ of these modules, and trying to understand every one of them well enough to do such a conversion is a path to madness, with little benefit. So, instead, Axboe started by implementing [a set of helpers](<https://lwn.net/ml/linux-kernel/20240411153126.16201-4-axboe@kernel.dk/>) that emulates the new functions with a series of calls to `read()` or `write()`; that minimizes the amount of change to any given module while maximizing the chances that the results will be correct. See [this patch](<https://lwn.net/ml/linux-kernel/20240411153126.16201-18-axboe@kernel.dk/>) as an example of what the simplest conversions look like. 

The [final patch](<https://lwn.net/ml/linux-kernel/20240411153126.16201-438-axboe@kernel.dk/>) in the series removes `read()` and `write()` with a surprising lack of ceremony, given that they have been there for 32 years. 

There have not been a lot of comments on the series; perhaps many developers are still waiting for the whole thing to download into their inboxes. Al Viro [noted](<https://lwn.net/ml/linux-kernel/20240412042910.GK2118490@ZenIV/>) that some of the conversions might need to be done a bit more carefully. But nobody has objected to the overall concept, thus far. 

For a series like this to be accepted, it will need to be split into more manageable chunks — which Axboe acknowledged at the outset. This set of changes does simplify the kernel, though, and it removes a fair amount of old code, so chances are that it will happen in some form, sooner or later. At that point, there will likely be a lot of out-of-tree modules that will need to be updated before they can be built on newer kernels. The good news is that developers can make those changes now and get ahead of the game.
