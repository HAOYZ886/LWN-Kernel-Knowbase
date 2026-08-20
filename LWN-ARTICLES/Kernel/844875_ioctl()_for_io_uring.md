---
title: ioctl() for io_uring
url: https://lwn.net/Articles/844875/
date: "February 4, 2021"
category: io uring; ioctl
author: "By Jonathan Corbet February 4, 2021"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
February 4, 2021

Of all the system calls in the Unix tradition, few are as maligned as [`ioctl()`](<https://man7.org/linux/man-pages/man2/ioctl.2.html>). But `ioctl()` exists for a reason — for many reasons, in truth — and cannot be expected to go away anytime soon. It is thus unsurprising that there is interest in providing `ioctl()`-like functionality in the [io_uring subsystem](<https://lwn.net/Articles/776703/>). A recent [RFC patch set from Jens Axboe](<https://lwn.net/ml/io-uring/20210127212541.88944-1-axboe@kernel.dk/>) shows the form that this feature might take in the io_uring context. 

The `ioctl()` name comes from "I/O control"; this system call was added as a way of performing operations on peripheral devices that went beyond reading and writing data. It could be used to rewind a tape drive, set the baud rate of a serial port, or eject a removable disk, for example. Over the years, uses of `ioctl()` have grown far beyond such simple applications, with some APIs ([media](<https://www.kernel.org/doc/html/latest/userspace-api/media/index.html>), for example) providing hundreds of operations. 

The criticism of `ioctl()` comes from its multiplexed and device-dependent nature; almost anything that can be represented by a file descriptor supports `ioctl()`, but the actual operations supported vary from one to the next. While system calls are (in theory, at least) closely scrutinized before being added to the kernel, `ioctl()` commands often receive close to no review at all. So nobody really knows everything that can be done with `ioctl()`. For added fun, there is some overlap in the command space, meaning that an `ioctl()` call made to the wrong file descriptor could have unexpected and highly unpleasant results. Attempts have been made to avoid this problem, but they have not been completely successful. 

After dealing with these problems for years, some developers would like to see `ioctl()` disappear completely, but nobody has ever come up with a replacement that looks materially better. Adding a new system call for every function that might be implemented with `ioctl()` is a non-starter; having device drivers interpret command streams sent with `write()` is even worse. There probably is no better way to, for example, tell a camera sensor which color space to use. 

It is natural to want to support `ioctl()` in io_uring; it is not uncommon to mix `ioctl()` calls with regular I/O, and it would be useful to be able to do everything asynchronously. But every `ioctl()` call is different, and none of them were designed for asynchronous execution, so an `ioctl()` implementation within io_uring would have no choice but to execute every call in a separate thread. That might be better than nothing, but it is not anywhere near as efficient as it could be, especially for calls that can be executed right away. Doing `ioctl()` right for io_uring essentially calls for reinventing the `ioctl()` interface. 

Operations in io_uring are communicated from user space to the kernel via a ring buffer; each is represented as an instance of the somewhat complex [`io_uring_sqe` structure](<https://elixir.bootlin.com/linux/v5.10.12/source/include/uapi/linux/io_uring.h#L13>). The new command mechanism is invoked by setting `opcode` in that structure to `IORING_OP_URING_CMD`; the `fd` field must, as usual, contain the file descriptor to operate on. The rest of the structure, though (starting with the `off` field) is overlaid with something completely different: 

```
struct io_uring_pdu {
    	__u64 data[4];	/* available for free use */
    	__u64 reserved;	/* can't be used by application! */
    	__u64 data2;	/* available for free use */
        };
```

The `reserved` field overlays `user_data` in the original structure, which is needed for other purposes; thus, no data relevant to the command can be stored there. Applications are unlikely to see this structure, though; it will be overlaid yet again with a structure specific to the command to be executed. For block-subsystem commands, for example, this structure becomes: 

```
struct block_uring_cmd {
    	__u16 	op;
    	__u16	pad;
    	union {
    	    __u32	size;
    	    __u32	ioctl_cmd;
    	};
    	__u64	addr;
    	__u64	unused[2];
    	__u64	reserved;	/* can never be used */
    	__u64	unused2;
        };
```

Deep down within this structure is `ioctl_cmd`, which the application should set to the `ioctl()` command code of interest; the `op` field should be `BLOCK_URING_OP_IOCTL` (for now; in the future there could be operations that are not tied to an `ioctl()` call). In the patch set, the only supported command is `BLKBSZGET`, which returns the block size of the underlying block device — something that can clearly be done without performing actual I/O or sleeping. The patch set also [implements a couple of networking commands](<https://lwn.net/ml/io-uring/20210127212541.88944-6-axboe@kernel.dk/>) using a different structure. 

Within the kernel, any subsystem that wants to support io_uring operations must add yet another field to the forever-growing [`file_operations` structure](<https://elixir.bootlin.com/linux/v5.10.12/source/include/linux/fs.h#L1822>): 

```
struct io_uring_cmd {
    	struct file *file;
    	struct io_uring_pdu pdu;
    	void (*done)(struct io_uring_cmd *, ssize_t);
        };
    
        int (*uring_cmd)(struct io_uring_cmd *, enum io_uring_cmd_flags);
```

Needless to say, any handlers for io_uring `IORING_OP_URING_CMD` operations should not block. Instead, they can complete the operation immediately, return an error indicating that the operation would block, or run the operation asynchronously and signal completion by calling the given `done()` function. 

This is an initial posting of a change that could have long-term implications, so it would not be surprising to see significant changes before it makes it into the mainline. Indeed, in response to [a comment from Darrick Wong](<https://lwn.net/ml/io-uring/20210128003831.GE7695@magnolia/>), Axboe [tweaked the interface](<https://lwn.net/ml/io-uring/f8576940-5441-1355-c09e-db60ad0ac889@kernel.dk/#t>) to provide eight more bytes of space in `struct io_uring_pdu` — something that Wong said would be highly useful to be able to submit the ""millions upon millions of ioctl calls"" created by the `xfs_scrub` utility. 

Whether the addition of an `ioctl()`-like interface to io_uring — which is rapidly evolving into a sort of shadow, asynchronous system-call interface for Linux — will generate controversy remains to be seen; there has been none in response to the initial posting. Axboe [expressed hope](<https://lwn.net/ml/io-uring/20210127212541.88944-2-axboe@kernel.dk/>) that the new commands will be ""a lot more sane and useful"" than the existing `ioctl()` commands, but there doesn't seem to be any way to enforce that. As with `ioctl()`, the addition of new io_uring commands will happen entirely within other subsystems, and the level of scrutiny those additions receive will vary. But io_uring needs this sort of "miscellaneous command" capability in the same way that the system as a whole needs `ioctl()`, so it would be surprising if this feature were not eventually merged in some form.
