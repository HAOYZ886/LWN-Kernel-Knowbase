---
title: Ringing in a new asynchronous I/O API
url: https://lwn.net/Articles/776703/
date: "January 15, 2019"
category: Asynchronous IO; io uring
author: "By Jonathan Corbet January 15, 2019"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
January 15, 2019

While the kernel has had support for [asynchronous I/O](<http://man7.org/linux/man-pages/man2/io_submit.2.html>) (AIO) since the 2.5 development cycle, it has also had people complaining about AIO for about that long. The current interface is seen as difficult to use and inefficient; additionally, some types of I/O are better supported than others. That situation may be about to change with the introduction of a [proposed new interface](<https://lwn.net/ml/linux-fsdevel/20190112213011.1439-1-axboe@kernel.dk/>) from Jens Axboe called "io_uring". As might be expected from the name, io_uring introduces just what the kernel needed more than anything else: yet another ring buffer. 

#### Setting up

Any AIO implementation must provide for the submission of operations and the collection of completion data at some future point in time. In io_uring, that is handled through two ring buffers used to implement a submission queue and a completion queue. The first step for an application is to set up this structure using a new system call: 

```
int io_uring_setup(int entries, struct io_uring_params *params);
```

The `entries` parameter is used to size both the submission and completion queues. The `params` structure looks like this: 

```
struct io_uring_params {
    	__u32 sq_entries;
    	__u32 cq_entries;
    	__u32 flags;
    	__u16 resv[10];
    	struct io_sqring_offsets sq_off;
    	struct io_cqring_offsets cq_off;
        };
```

On entry, this structure (with the possible exception of `flags` as described later) should simply be initialized to zero. On return from a successful call, the `sq_entries` and `cq_entries` fields will be set to the actual sizes of the submission and completion queues; the code is set up to allocate `entries` submission entries, and twice that many completion entries. 

The return value from `io_uring_setup()` is a file descriptor that can then be passed to `mmap()` to map the buffer into the process's address space. More specifically, three calls are needed to map the two ring buffers and an array of submission-queue entries; the information needed to do this mapping will be found in the `sq_off` and `cq_off` fields of the `io_uring_params` structure. In particular, the submission queue, which is a ring of integer array indices, is mapped with a call like: 

```
subqueue = mmap(0, params.sq_off.array + params.sq_entries*sizeof(__u32),
        		    PROT_READ|PROT_WRITE|MAP_SHARED|MAP_POPULATE,
    		    ring_fd, IORING_OFF_SQ_RING);
```

Where `params` is the `io_uring_params` structure, and `ring_fd` is the file descriptor returned from `io_uring_setup()`. The addition of `params.sq_off.array` to the length of the region accounts for the fact that the ring is not located right at the beginning. The actual array of submission-queue entries, instead, is mapped with: 

```
sqentries = mmap(0, params.sq_entries*sizeof(struct io_uring_sqe),
        		    PROT_READ|PROT_WRITE|MAP_SHARED|MAP_POPULATE,
    		    ring_fd, IORING_OFF_SQES);
```

This separation of the queue entries from the ring buffer is needed because I/O operations may well complete in an order different from the submission order. The completion queue is simpler, since the entries are not separated from the queue itself; the incantation required is similar: 

```
cqentries = mmap(0, params.cq_off.cqes + params.cq_entries*sizeof(struct io_uring_cqe),
        		    PROT_READ|PROT_WRITE|MAP_SHARED|MAP_POPULATE,
    		    ring_fd, IORING_OFF_CQ_RING);
```

It's perhaps worth noting at this point that Axboe is working on a user-space library that will hide much of the complexity of this interface from most users. 

#### I/O submission

Once the io_uring structure has been set up, it can be used to perform asynchronous I/O. Submitting an I/O request involves filling in an `io_uring_sqe` structure, which looks like this (simplified a bit): 

```
struct io_uring_sqe {
    	__u8	opcode;		/* type of operation for this sqe */
    	__u8	flags;		/* IOSQE_ flags */
    	__u16	ioprio;		/* ioprio for the request */
    	__s32	fd;		/* file descriptor to do IO on */
    	__u64	off;		/* offset into file */
    	void	*addr;		/* buffer or iovecs */
    	__u32	len;		/* buffer size or number of iovecs */
    	union {
    	    __kernel_rwf_t	rw_flags;
    	    __u32		fsync_flags;
    	};
    	__u64	user_data;	/* data to be passed back at completion time */
    	__u16	buf_index;	/* index into fixed buffers, if used */
        };
```

The `opcode` describes the operation to be performed; options include `IORING_OP_READV`, `IORING_OP_WRITEV`, `IORING_OP_FSYNC`, and a couple of others that we will return to. There are clearly a number of parameters that affect how the I/O is performed, but most of them are relatively straightforward: `fd` describes the file on which the I/O will be performed, for example, while `addr` and `len` describe a set of `iovec` structures pointing to the memory where the I/O is to take place. 

As mentioned above, the `io_uring_sqe` structures are kept in an array that is mapped into both user and kernel space. Actually submitting one of those structures requires placing its index into the submission queue, which is defined this way: 

```
struct io_uring {
    	u32 head;
    	u32 tail;
        };
    
        struct io_sq_ring {
    	struct io_uring		r;
    	u32			ring_mask;
    	u32			ring_entries;
    	u32			dropped;
    	u32			flags;
    	u32			array[];
        };
```

The `head` and `tail` values are used to manage entries in the ring; if the two values are equal, the ring is empty. User-space code adds an entry by putting its index into `array[r.tail]` and incrementing the tail pointer; only the kernel side should change `r.head`. Once one or more entries have been placed in the ring, they can be submitted with a call to: 

```
int io_uring_enter(unsigned int fd, u32 to_submit, u32 min_complete, u32 flags);
```

Here, `fd` is the file descriptor associated with the ring, and `to_submit` is the number of entries in the ring that the kernel should submit at this time. The return value should be zero if all goes well. 

Completion events will find their way into the completion queue as operations are executed. If `flags` contains `IORING_ENTER_GETEVENTS` and `min_complete` is nonzero, `io_uring_enter()` will block until at least that many operations have completed. The actual results can be found in the completion structure: 

```
struct io_uring_cqe {
    	__u64	user_data;	/* sqe->user_data submission passed back */
    	__s32	res;		/* result code for this event */
    	__u32	flags;
        };
```

Where `user_data` is a value passed from user space when the operation was submitted and `res` is the return code for the operation. The `flags` field will contain `IOCQE_FLAG_CACHEHIT` if the request could be satisfied without needing to perform I/O — an option that may yet have to be reconsidered given the current concern about using the page cache as a side channel. 

These structures live in the completion queue, which looks similar to the submission queue: 

```
struct io_cq_ring {
    	struct io_uring		r;
    	u32			ring_mask;
    	u32			ring_entries;
    	u32			overflow;
    	struct io_uring_cqe	cqes[];
        };
```

In this ring, the `r.head` index points to the first available completion event, while `r.tail` points to the last; user space should only change `r.head`. 

The interface as described so far is enough to enable a user-space program to enqueue multiple I/O operations and to collect the results as those operations complete. The functionality is similar to what the current AIO interface provides, though the interface is quite different. Axboe claims that it is far more efficient, but no benchmark results have been included yet to back up that claim. Among other things, this interface can do asynchronous buffered I/O without a context switch in cases where the desired data is in the page cache; buffered I/O has always been a bit of a sore spot for Linux AIO. 

#### Advanced features

There are, however, some more features worthy of note in this interface. One of those is the ability to map a program's I/O buffers into the kernel. This mapping normally happens with each I/O operation so that data can be copied into or out of the buffers; the buffers are unmapped when the operation completes. If the buffers will be used many times over the course of the program's execution, it is far more efficient to map them once and leave them in place. This mapping is done by filling in yet another structure describing the buffers to be mapped: 

```
struct io_uring_register_buffers {
    	struct iovec *iovecs;
    	__u32 nr_iovecs;
        };
```

That structure is then passed to another new system call: 

```
int io_uring_register(unsigned int fd, unsigned int opcode, void *arg);
```

In this case, the opcode should be `IORING_REGISTER_BUFFERS`. The buffers will remain mapped for as long as the initial file descriptor remains open, unless the program explicitly unmaps them with `IORING_UNREGISTER_BUFFERS`. Mapping buffers in this way is essentially locking memory into RAM, so the usual resource limit that applies to `mlock()` applies here as well. When performing I/O to premapped buffers, the `IORING_OP_READ_FIXED` and `IORING_OP_WRITE_FIXED` operations should be used. 

There is also an `IORING_REGISTER_FILES` operation that can be used to optimize situations where many operations will be performed on the same file(s). 

In many high-bandwidth settings, it can be more efficient for the application to poll for completion events rather than having the kernel collect them and wake the application up; that is the motivation behind the [existing block-layer polling interface](<https://lwn.net/Articles/663879/>), for example. Polling is most efficient in situations where, by the time the application gets around to doing a poll, there is almost certainly at least one completion ready for it to consume. This polling mode can be enabled for io_uring by setting the `IORING_SETUP_IOPOLL` flag when calling `io_uring_setup()`. In such rings, an occasional call to `io_uring_enter()` (with the `IORING_ENTER_GETEVENTS` flag set) is mandatory to ensure that completion events actually make it into the completion queue. 

Finally, there is also a fully polled mode that (almost) eliminates the need to make any system calls at all. This mode is enabled by setting the `IORING_SETUP_SQPOLL` flag at ring setup time. A call to `io_uring_enter()` will kick off a kernel thread that will occasionally poll the submission queue and automatically submit any requests found there; receive-queue polling is also performed if it has been requested. As long as the application continues to submit I/O and consume the results, I/O will happen with no further system calls. 

Eventually, though (after one second currently), the kernel will get bored if no new requests are submitted and the polling will stop. When that happens, the `flags` field in the submission queue structure will have the `IORING_SQ_NEED_WAKEUP` bit set. The application should check for this bit and, if it is set, make a new call to `io_uring_enter()` to start the mechanism up again. 

This patch set is in its third version as of this writing, though that is a bit deceptive since there were (at least) ten revisions of the [polled AIO patch set](<https://lwn.net/ml/linux-fsdevel/20181221192236.12866-1-axboe@kernel.dk/>) that preceded it. While it is possible that the interface is beginning to stabilize, it would not be surprising to see some significant changes yet. One review comment that has not yet been addressed is Matthew Wilcox's [request](<https://lwn.net/ml/linux-fsdevel/20190109160036.GK6310@bombadil.infradead.org/>) that the name be changed to ""something that looks a little less like io_urine"". That could yet become the biggest remaining issue — as we all know, naming is always the hardest part in the end. But, once those details are worked out, the kernel may yet have an asynchronous I/O implementation that is not a constant source of complaints. 

For the curious, Axboe has posted [a complete example](<http://git.kernel.dk/cgit/fio/plain/t/io_uring.c>) of a program that uses the io_uring interface.
