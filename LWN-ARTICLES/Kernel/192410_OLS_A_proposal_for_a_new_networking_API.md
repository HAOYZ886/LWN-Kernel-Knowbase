---
title: "OLS: A proposal for a new networking API"
url: https://lwn.net/Articles/192410/
date: "July 22, 2006"
category: Asynchronous IO; Events reporting; Kevent; Networking
author: 
---

Ulrich Drepper has been the maintainer of the core glibc library since 1995; he also represents the community to the POSIX standardization effort. So, when Ulrich proposes a new user-space API, more than the usual number of people are likely to listen. Ulrich has been putting his mind to the problems of high-performance network I/O; the results were presented at his Ottawa Linux Symposium talk. 

The current POSIX APIs are, increasingly, not up to the task. The socket abstraction has served us for a long time, but it is a synchronous interface which is not well suited to zero-copy I/O. POSIX does provide an asynchronous I/O interface, but it was never intended for use with networking, and does not provide the requisite functionality. So it has been clear for a while that something better is needed; the developers working on [network channels](<http://lwn.net/Articles/169961/>) have also been talking about the need for a new networking API. 

There are three components to a new networking API, all of which will lead to a more complex - but much more efficient - interface for high-performance situations. The first of those is to address the need for zero-copy I/O. As the data bandwidth through the system increases, the cost of copying data (in CPU utilization and cache pressure) increase. Much of this cost can be avoided by transferring data directly between the network interface and buffers in user space. Direct user-space I/O requires cooperation from both the kernel and the application, however. 

Ulrich proposes the creation of an interface for the explicit management of user-space DMA areas. Such an area would be created with a call that looks something like: 

```
int dma_alloc(dma_mem_t *handle, size_t size, int flags);
```

If all goes well, the result would be a memory area of the given `size`, suitable for DMA purposes. Note that user space gets an opaque handle type in return - there is, at this point, no virtual address which is directly accessible to the application. 

To use a DMA area for network I/O, the application must associate it with a socket. The call for this operation would look like: 

```
int dma_assoc(int socket, dma_mem_t handle, size_t size, int flags);
```

There is still the issue of actually managing memory within this DMA area. An application which is generating data to send over the net would request a buffer from the kernel with a call like: 

```
int sio_reserve(dma_mem_t handle, void **buffer, size_t size);
```

If all goes well, the result will be a pointer (stored in `*buffer`) to an area where the outgoing data can be constructed. For incoming data, the application will receive a pointer to the buffer from the kernel (just how is something we'll get to shortly); the application will own the given buffer until it returns it to the kernel with: 

```
int sio_release(dma_mem_t handle, size_t size);
```

Before an application can start to use asynchronous network I/O, however, it must have a way to learn about the results of its operations. To that end, Ulrich proposes the addition of an event reporting API to the kernel. This mechanism, which he calls "event channels," would have an interface like: 

```
ec_t ec_create(int flags); /* Create a channel */
        ec_next_event();           /* Get the next event */
        ec_to_fd();                /* Send events to a file descriptor */
        ec_delay();                /* Wait for an event directly */
```

The exact form of this interface (like all of those discussed here) is subject to change. But the core idea is that it is a quick way for the kernel to return notifications of events (such as I/O completions) to user space. Most applications would be likely to use the file descriptor interface, which would allow events to enter an application's main loop via `poll()` or `epoll_wait()`. 

The final step is to make some extensions to the existing POSIX asynchronous I/O interface. The `aiocb` structure would be extended to include an event channel descriptor; that channel would be used to report the results of asynchronous operations back to user space. Then, an application could initiate data transmission with a call like: 

```
int aio_send(int socket, void *buffer, size_t size, int flags);
```

(One presumes there would be an `aiocb` argument as well, but Ulrich's slides did not show one). This call would start the process of transmitting data from the given `buffer`, with completion likely happening sometime after the call returns. For data reception, the call would look like: 

```
int aio_recv(int socket, void **buffer, size_t size, int flags);
```

The relevant point here being that `buffer` is a double pointer; the kernel would pick the actual destination for the data and tell the calling application where to look. 

The result of all these changes would be a complete programming interface for high-performance, asynchronous network I/O. As an added bonus, the use of an event channel interface would simplify the work of porting applications from other operating systems. 

All of these interfaces, says Ulrich, are simply a proposal and subject to massive change. The core purpose is to allow applications to get their work done while giving the kernel the greatest possible latitude to optimize the data transfers. This proposal is not the only one out there; Evgeniy Polyakov's [kevent proposal](<http://lwn.net/Articles/172844/>) is similar in many ways, though it does not have the explicit management of DMA areas. It may be some time before something is actually adopted - a new API will stay around for many years and should not be added in haste - but the discussion is getting started in earnest.
