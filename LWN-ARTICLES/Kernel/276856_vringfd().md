---
title: vringfd()
url: https://lwn.net/Articles/276856/
date: "April 7, 2008"
category: Events reporting; vringfd
author: "By Jonathan Corbet April 7, 2008"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
April 7, 2008

One of the core features of the (now stalled) kevent subsystem was a circular buffer intended for efficient movement of data between the kernel and user space. Kevent may have run out of steam, but the ring buffer idea is back via a different path. Rusty Russell is now [proposing](<http://lwn.net/Articles/276729/>) a new system call (called `vringfd()`) which turns some of the [virtio](<http://lwn.net/Articles/239238/>) work into a new kernel-to-user ring buffer interface. The submitted patch is breathtaking in its lack of documentation on this new system call, especially considering that its author is quite good with that sort of writing. Your editor has taken this omission as a personal challenge and, as a result, has set about reverse engineering the (somewhat complex) `vringfd()` interface. 

A user-space process which wishes to set up a vring for communication with the kernel must create a slightly complicated data structure first. One starts by deciding how many entries the ring should have; this number must be a power of two which fits into an unsigned, 16-bit value. Given this number (we'll call it `RING_SIZE`), the data structure looks like this: 

```
struct messy_vring_thing {
    	struct vring_desc descriptors[RING_SIZE];
    	struct vring_avail available;
    	char padding[up-to-next-page-boundary];
    	struct vring_used used[RING_SIZE];
        };
```

The page alignment for the `used` array is important - that array might be mapped separately into kernel space. The array must fit into a single page, which puts a practical limit of 256 entries for `RING_SIZE` on systems with 4096-byte pages. If this API goes forward, chances are good that a way will be found to raise this limit. 

Individual descriptors in the ring are described with this structure: 

```
struct vring_desc
        {
    	__u64 addr;	/* Address of the buffer */
    	__u32 len; 	/* Length of the buffer */
    	__u16 flags;	/* some flags */
    	__u16 next;	/* Next buffer in the chain */
        };
```

For a simple buffer, the application would simply point `addr` at the beginning and set `len` to the appropriate value. If the buffer is to be written to by the kernel, the application should also set `VRING_DESC_F_WRITE` in the `flags` field. 

Things can get more complicated than that, though, in that the `vringfd()` interface supports multipart scatter/gather buffers. To set up such a buffer, user space would use one `vring_desc` entry for each segment of the buffer. For all but the final segment, the `VRING_DESC_F_NEXT` flag (saying "use the next descriptor too") should be set, and `next` should be the index of the next descriptor. When the kernel grabs a buffer, it will follow the chain and use all segments found until the final one (which lacks the `VRING_DESC_F_NEXT` flag) is encountered. 

Before the kernel will use buffers set up by the application, though, user space must indicate that the buffer is ready. That is done through the `vring_avail` structure: 

```
struct vring_avail
        {
    	__u16 flags;
    	__u16 idx;
    	__u16 ring[RING_SIZE];
        };
```

The `ring` array holds indexes into the `descriptors` array. The `idx` field should always be the index of the last valid entry in `ring`. When a new buffer is ready for transfer to or from the kernel, the application will store the index of the first descriptor into `ring[idx+1]`, then increment `idx`. When the ring is first established, the kernel remembers the position of `idx`, so the first buffer should be added here _after_ the `vringfd()` system call is made. 

The kernel will consume buffers from the `available` ring as needed. Once the requested operation has been performed on the buffer and the kernel is done with it, the buffer will show up in the `used` area, which is structured this way: 

```
struct vring_used_elem
        {
    	__u32 id;
    	__u32 len;
        };
    
        struct vring_used
        {
    	__u16 flags;
    	__u16 idx;
    	struct vring_used_elem ring[RING_SIZE];
        };
```

In the `vring_used` structure, `idx` is the index of the next entry in `ring` which may be written by the kernel; it will be incremented after the ring is updated. When a buffer is placed in the used ring, the `id` field will be the index of the descriptor, and `len` will be the actual length of the data transferred. 

Note that the `flags` fields in the `vring_avail` and `vring_used` structures appear to be unused. 

Once the application has this whole data structure set up, it can establish the ring buffer with the kernel with the new system call: 

```
long vringfd(void *addr, unsigned int ring_size, u16 *last_used);
```

Here, `addr` is the base address of the data structure described above, `ring_size` is the number of descriptors in the ring, and `last_used` is a 16-bit unsigned integer indicating which entry in the `used` ring was last consumed by the application. Failure to keep `last_used` current will not slow things down, but it will keep `poll()` from working properly. 

The return value will be a file descriptor associated with the ring. 

Creating the vring is only part of the job, though. The next step is to connect it with a kernel subsystem for the transfer of data. Rusty's patch includes vring support in the tun virtual network driver; to use that support, an application makes a special `ioctl()` call to provide the vring file descriptor to the tun driver. Any other subsystem will need a similar mechanism to support vring. 

If the application is using the ring to transfer data into the kernel, it must (1) set up one or more descriptors for full data buffers in the `available` ring, then (2) make a `write()` call to the vring file descriptor. The buffer and length passed to `write()` are ignored; all that matters is that a write was done to that file descriptor. When `write()` returns the operation will have been set in motion, but it cannot be considered to be complete until the ring descriptors show up in the `used` ring. 

For data transfers from the kernel to user space, the application simply puts buffers into the `available` ring, then waits until they show up in the `used` ring. A `poll()` on the vring file descriptor will block until buffers are available. The kernel determines whether unconsumed buffers exist in `used` by comparing the `vring_used->idx` index against the application-supplied `last_used` value. It's worth noting that, depending on how the relevant kernel subsystem works, buffers may not actually make it into the `used` ring until the `poll()` call is made. 

On the kernel side, a developer wanting to add vring support to a subsystem will start by creating a set of `vring_ops`: 

```
struct vring_ops
        {
    	void (*destroy)(void *);
    	int (*pull)(void *);
    	int (*push)(void *);
        };
```

All of these functions take a private pointer given when the subsystem attaches to the vring (to be described shortly). The `pull()` callback is invoked when the application calls `poll()`; if there is any descriptor processing which must be done with user space accessible, this is the place to do it. If `pull()` adds any buffers to the `used` ring, it should return the number of buffers; it can also return a negative error code. `push()` is called from a `write()` call indicating that there are buffers ready to be transferred into the kernel; it returns zero or a negative error code. The `destroy()` callback is called when the vring file descriptor is closed. All of these callbacks are optional. 

Attaching to a vring is done with: 

```
struct vring_info *vring_attach(int fd, const struct vring_ops *ops,
    				    void *data, bool atomic_use);
```

For this call, `fd` is a file descriptor corresponding to a vring, `ops` is the operations structure described above, `data` is a private data pointer which is passed into the `vring_ops` callbacks, and `atomic_use` is nonzero if the kernel needs to be able to add buffers to the `used` ring in atomic context. The return value is a pointer to an internal vring data structure or an `ERR_PTR()` value if something goes wrong. 

To obtain a buffer from the `available` ring, a call is made to: 

```
int vring_get_buffer(struct vring_info *vr,
    		         struct iovec *in_iov,
    		     	 unsigned int *num_in, unsigned long *in_len,
    		     	 struct iovec *out_iov,
    		     	 unsigned int *num_out, unsigned long *out_len);
```

This function will fill in an array of `iovec` structures corresponding to the next available buffer. If the kernel expects to write to the buffer, it should set `in_iov` to the `iovec` array, `num_in` pointing to the length of `in_iov`, and `in_len` pointing to a location to store the total length of the buffer (or `NULL` if that information is not useful). For transfers into the kernel, `out_iov`, `num_out`, and `out_len` should be set similarly. Note that the addresses stored in the `iovec` arrays are user-space addresses; `vring_get_buffer()` does not validate them, so the caller must do so. 

It is possible to set pass both `in_iov` and `out_iov`; in this case, one of the two will be set, depending on whether the next buffer in the `available` ring has the `VRING_DESC_F_WRITE` flag set. In most cases, though, only one of the two sets of parameters will have non-`NULL` values. The apparent intent of the API is that, if bidirectional transfers between user space and the kernel are needed, two separate vrings should be used. 

The return value from `vring_get_buffer` will be one of (1) a positive descriptor index, (2) zero, indicating that no buffers are available, or (3) a negative error code. 

The descriptor index should be saved the the final step, which is indicating that the kernel is done with a specific buffer: 

```
void vring_used_buffer(struct vring_info *vr, int id, u32 len);
        void vring_used_buffer_atomic(struct vring_info *vr, int id, u32 len);
```

Either one of these functions indicates that the buffer indicated by `id` should be put into the `used` ring; `len` is the amount of data actually transferred. If sleeping is not possible, `vring_used_buffer_atomic()` should be used - but the vring must have been attached with the `atomic_use` flag set. 

There does not appear to be a way for a subsystem to detach from a vring; it must, instead, wait for the application to close the associated file descriptor. 

This interface is in an early stage, and the code has a number of limitations and FIXME comments. So things seem likely to evolve before `vringfd()` is seriously considered for merging into the mainline kernel. The idea of a ring buffer for this kind of communication seems to come around on a regular basis, though, so it would seem that there is a demand for this kind of API.
