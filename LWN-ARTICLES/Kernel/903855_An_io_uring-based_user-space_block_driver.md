---
title: "An io_uring-based user-space block driver"
url: https://lwn.net/Articles/903855/
date: "August 8, 2022"
category: "Block layer-Block drivers; Releases-6.0; io uring"
author: "By Jonathan Corbet August 8, 2022"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
August 8, 2022

The addition of the ublk driver during the 6.0 merge window would have been easy to miss; it was buried deeply within an io_uring pull request and is entirely devoid of any sort of documentation that might indicate why it merits a closer look. Ublk is intended to facilitate the implementation of high-performance block drivers in user space; to that end, it uses [io_uring](<https://lwn.net/Articles/776703/>) for its communication with the kernel. This driver is considered experimental for now; if it is successful, it might just be a harbinger of more significant changes to come to the kernel in the future. 

Your editor has spent a fair amount of time beating his head against [the source for the ublk driver](<https://lwn.net/ml/io-uring/20220713140711.97356-1-ming.lei@redhat.com/>), as well as the [ubdsrv server](<https://github.com/ming1/ubdsrv>) that comprises the user-space component. The picture that has emerged from this exploration of that uncommented and vowel-deficient realm is doubtless incorrect in some details, though the overall shape should be close enough to reality. 

#### How ublk works

The ublk driver starts by creating a special device called `/dev/ublk-control`. The user-space server (or servers, there can be more than one) starts by opening that device and setting up an io_uring ring to communicate with it. Operations at this level are essentially `ioctl()` commands, but `/dev/ublk-control` has no `ioctl()` handler; all operations are, instead, sent as commands through io_uring. Since the purpose is to implement a device behind io_uring, the reasoning seems to be, there is no reason to not use it from the beginning. 

A server will typically start with a `UBLK_CMD_ADD_DEV` command; as one might expect, it adds a new ublk device to the system. The server can describe various aspects of this device, including the number of hardware queues it claims to implement, its block size, the maximum transfer size, and the number of blocks the device can hold. Once this command succeeds, the device exists as far as the ublk driver is concerned and is visible as `/dev/ublkc _N_` , where `_N_` is the device ID returned when the device is created. The device has not yet been added to the block layer, though. 

The server should open the new `/dev/ublkc _N_` device for the following steps, the first of which is to map a region from the device into the server's address space with an `mmap()` call. This region is an array of `ublksrv_io_desc` structures describing I/O requests: 

```
struct ublksrv_io_desc {
    	/* op: bit 0-7, flags: bit 8-31 */
    	__u32		op_flags;
    	__u32		nr_sectors;
    	__u64		start_sector;
    	__u64		addr;
        };
```

Notification of new I/O requests will be received via io_uring. To get to that point, the server must enqueue a set of `UBLK_IO_FETCH_REQ` requests on the newly created device; normally there will be one for each "hardware queue" declared for the device, which may also correspond to each thread running within the server. Among other things, this request must provide a memory buffer that can hold the maximum request size declared when the device was created. 

Once this setup is complete, a separate `UBLK_CMD_START_DEV` operation will cause the ublk driver to actually create a block device visible to the rest of the system. When the block subsystem sends a request to this device, one of the queued `UBLK_IO_FETCH_REQ` operations will complete. The completion data returned to the user-space server will include the index of the `ublkserv_io_desc` structure describing the request, which the server should now execute. For a write request, the data to be written will be in the buffer that was provided by the server; for a read, the data should be placed in that same buffer. 

When the operation is complete, the server must inform the kernel of that fact; this is done by placing a `UBLK_IO_COMMIT_AND_FETCH_REQ` operation into the ring. It will give the result of the operation back to the block subsystem, but will also enqueue the buffer to receive the next request, thus avoiding the need to do that separately. 

There are the expected `UBLK_CMD_STOP_DEV` and `UBLK_CMD_DEL_DEV` operations to make existing devices go away, and a couple of other operations to query information about existing devices. There are also a number of details that have not been covered here, mostly aimed at increased performance. Among other things, the ublk protocol is set up to enable zero-copy I/O, but that is not implemented in the current code. 

The server code implements two targets: null and loop. The null target is, as one might expect, an overly complicated, block-oriented version of `/dev/null`; it is useless but makes it possible to see how things work with a minimum of unrelated details. The loop target uses an existing file as the backing store for a virtual block device. According to author Ming Lei, with this loop implementation, ""the performance is is even better than kernel loop with same setting"". 

#### Implications

One might wonder why this work has been done (and evidently supported by Red Hat); if the world has been clamoring for an io_uring-based, user-space, faster loop block device, it has done so quietly. One advantage cited in the patch cover letter is that development of block-driver code is more easily done in user space; another is high-performance [qcow2](<https://en.wikipedia.org/wiki/Qcow>) support. The patch cover letter also cites interest expressed by other developers in having a fast user-space block-device mechanism available. 

An interesting question, though, is whether this mechanism might ultimately facilitate the movement of a number of device drivers out of the kernel — perhaps not just block drivers. Putting device drivers into user-space code is a fundamental concept in a number of secure-system designs, including microkernel systems. But one of the problems with those designs has always been the communication overhead between the two components once they are no longer running within the same address space. Io_uring might just be a convincing answer to that problem. 

Should that scenario play out, kernels of the future could look significantly different from what we have today; they could be smaller, with much of the complicated logic running in separate, user-space components. Whether this is part of Lei's vision for ublk is unknown, and things may never get anywhere near that point. But ublk is clearly an interesting experiment that could lead to big changes down the line. Something will need to be done about that complete absence of documentation, though, on the way toward world domination.
