---
title: A generic ring buffer for the kernel
url: https://lwn.net/Articles/976836/
date: "June 6, 2024"
category: "System calls-ringbuffer"
author: "By Jonathan Corbet June 6, 2024"
---

By **Jonathan Corbet**  
June 6, 2024

The kernel's user-space ABI does not lack for ring buffers; they have been defined for subsystems like [BPF](<https://docs.kernel.org/6.6/bpf/ringbuf.html>), [io_uring](<https://lwn.net/Articles/776703/>), [perf](<https://docs.kernel.org/next/userspace-api/perf_ring_buffer.html>), and [tracing](<https://docs.kernel.org/trace/ring-buffer-map.html>), for example. Naturally, each of those ring buffers is unique, with no common interface between them. The natural response to this ABI proliferation is, of course, to add yet another ring buffer as the generic option; that is the intent of [this patch series](<https://lwn.net/ml/linux-kernel/20240603003306.2030491-1-kent.overstreet@linux.dev>) from Kent Overstreet adding a new set of system calls for ring buffers. 

A ring buffer is simply a circular buffer, maintained in memory, that is shared between user space and the kernel. One side of a data stream writes data into the buffer, while the other consumes it; as long as the buffer neither overflows nor underflows, data can be transferred with no system calls at all. The addition of a ring buffer can, thus, enable highly efficient data transfer in situations where the data rates are relatively high. Overstreet thinks that other kernel subsystems could benefit from ring-buffer interfaces, and would like to make it possible to add those interfaces without reinventing the wheel. 

#### The user-space interface

On the user-space side, with the proposed interface, a ring buffer is always associated with a file descriptor, so the first step will be to open the object (such as a file, a device, or a pipe) to which the ring buffer will be attached. Then, the ring buffer is established with the new system call: 

```
int ringbuffer(unsigned int fd, int rw, u32 size, void **buffer);
```

This call will request the creation of a ring buffer associated with the open file descriptor `fd`; the `rw` should be set to zero for read access, or anything else for write. The requested size of the buffer is passed in `size`. On a successful return, the address for the buffer will be stored in `buffer`. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Note that it is up to the subsystem implementing the ring buffer to decide whether to respond to this call by creating a new buffer, or by mapping an existing buffer into the caller's address space. The intent in the latter case is to allow a single ring buffer to be shared between multiple processes. 

There is no `flags` argument in this version of the system call; that seems likely to change before any eventual merging into the mainline. 

The returned pointer will be aimed at a region of memory starting with a copy of this structure: 

```
struct ringbuffer_ptrs {
    	__u32		head;
    	__u32		tail;
    	__u32		size;	/* always a power of two */
    	__u32		mask;	/* size - 1 */
    	__u32		data_offset;
        };
```

The buffer itself will begin at `data_offset` from the returned pointer; in practice, that offset will likely always be one page. The actual size of the buffer will be stored in `size`; as noted, it will always be a power of two for reasons that will soon become clear. The `head` offset is the writing position, while `tail` is the read position. If `head` and `tail` are equal, the buffer is empty. 

Both `head` and `tail` are incremented unconditionally when data is added or consumed; there are no tests for wrapping around. Instead, these values must be ANDed with the `mask` value to obtain an actual offset into the buffer. This technique eliminates the need for branches in the increment path, but it also means that a little care is needed when working with the head and tail values. Testing for emptiness is easy, since the two values will always be equal in this case. A proper test for fullness looks like: 

```
if ((head - tail) >= size)
            /* buffer is full, can't write to it */
```

(This test has been simplified; a properly written test must use atomic operations to avoid memory-ordering problems; see [this test program](<https://lwn.net/ml/linux-kernel/20240603003306.2030491-6-kent.overstreet@linux.dev>) for an example). 

A process writing to a ring buffer can continue to feed data into it until the buffer fills, at which point it must stop. The writer could poll the `tail` value to detect when the reader has caught up and more space is available, but that would not be efficient. A similar situation applies to a reader that has emptied the buffer. Either way, the process can wait for the situation to change with: 

```
int ringbuffer_wait(unsigned int fd, int rw);
```

Where `fd` is the file descriptor associated with the ring buffer, and `rw` indicates whether read or write access is needed. Either way, the calling process will block until the situation with the ring changes. Indicating such a change is the responsibility of user space as well; a writer that has put data into a previously empty buffer, or a reader that has consumed data from a previously full buffer should call: 

```
int ringbuffer_wakeup(unsigned int fd, int rw);
```

In the cover letter, Overstreet suggests that `ringbuffer_wait()` and `ringbuffer_wakeup()` might eventually be replaced with futex operations. [This patch](<https://lwn.net/ml/linux-kernel/20240603003306.2030491-4-kent.overstreet@linux.dev>) also mentions the need to improve `ringbuffer_wait()` to allow for a timeout or to specify the minimum amount of data that must be written to (or read from) the buffer before a wakeup should happen. 

#### The kernel side

To add ring-buffer support to a subsystem, the new `ringbuffer()` function in the `file_operations` structure should be provided: 

```
struct ringbuffer *(*ringbuffer)(struct file *file, int rw);
```

This function will be called in response to a `ringbuffer()` call from user space. There is a whole set of meticulously undocumented support functions available to drivers for working with ring buffers, including `ringbuffer_alloc()` to create one, `ringbuffer_read_iter()` and `ringbuffer_write_iter()` to move data out of or into a ring buffer, etc. [This patch](<https://lwn.net/ml/linux-kernel/20240603003306.2030491-5-kent.overstreet@linux.dev>) provides a test driver showing the basics of implementing a ring buffer in kernel space. 

Other than the test device, there are no in-kernel users submitted with this patch set. In the cover letter, Overstreet says that the first such user will be the filesystems in user space (FUSE) subsystem, but pipes and sockets may follow thereafter. The performance benefits of the ring-buffer approach are said to be significant: ""reading/writing from the ringbuffer 16 bytes at a time is ~7x faster than using read/write syscalls"". 

As of this writing, there have been no responses to this submission. This work is in an early state and seems likely to need to evolve somewhat before it could be considered for merging; developers are also likely to want to see a real user in the kernel. Even more interesting might be a demonstration of how some of the existing kernel ring-buffer interfaces could have been implemented using this abstraction (not that they will change now). If this work achieves its goals, it might, in a highly optimistic view, be the last ring-buffer interface added to the kernel.
