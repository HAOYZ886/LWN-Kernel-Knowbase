---
title: The kevent interface
url: https://lwn.net/Articles/172844/
date: "February 22, 2006"
category: Asynchronous IO; Events reporting; Kevent
author: 
---

The Linux asynchronous I/O implementation is notoriously incomplete; among the many things on the "to do" list is asynchronous network I/O. Network writes are already, to some extent, asynchronous, but only if the kernel is able to copy user data into a kernel buffer. The current interface cannot be simultaneously zero-copy and asynchronous. There is also no way to set up asynchronous, zero-copy reads. Evgeniy Polyakov has recently [posted a patch](<http://lwn.net/Articles/172551/>) which tries to fill that gap - and quite a bit more besides - through the addition of three new system calls and a completely new kernel event subsystem. 

Evgeniy's patch adds a new "kevent" type. The kernel can generate and report kevents for a number of possible situations, including: 

  * The arrival of network data or connections. 
  * Any situation which can be reported by the `poll()` system call. 
  * Events which can be returned by `inotify()`, such as the creation or removal of files. 
  * Network asynchronous I/O events. 
  * Timer events. 

All of this becomes possible through the addition of a complex system call: 

```
struct kevent_user_control
        {
    	unsigned int cmd;
    	unsigned int num;
    	unsigned int timeout;
        };
    
        long kevent_ctl(int fd, struct kevent_user_control ctl);
```

The file descriptor argument to `kevent_ctl()` has little to do with any requested events; it is, instead, mostly used as a place for the kevent subsystem to stash some of its own housekeeping information. That file descriptor must be allocated, however, with a call like: 

```
ctl.cmd = KEVENT_CTL_INIT;
        int kevent_fd = kevent_ctl(0, &ctl);
```

The returned file descriptor can be used to add, remove, modify, and wait for events. Event requests are passed from user space in a structure like: 

```
struct kevent_id
        {
    	__u32		raw[2];
        };
    
        struct ukevent
        {
    	struct kevent_id id;
    	__u32 type;
    	__u32 event;
    	__u32 req_flags;
    	/* ... */
        };
```

Here, the embedded `id` structure usually holds a file descriptor number for which associated events are desired. For timer events, instead, it holds the timeout period. The `type` and `event` fields describe what sorts of events are desired; `type` can be one of: `KEVENT_SOCKET` (data and/or connections on sockets), `KEVENT_INODE` (file creation and removal), `KEVENT_POLL` (any `poll()` event), `KEVENT_TIMER` (timer events), or `KEVENT_NAIO` (network asynchronous I/O). The `event` field is a bitmask which depends on `type`; as an example, for inode events, it can contain `KEVENT_INODE_CREATE` and/or `KEVENT_INODE_REMOVE`. The main thing seen in `req_flags` is `KEVENT_REQ_ONESHOT`, indicating that only one event should be returned. 

The attentive reader may have noticed that the `kevent_ctl()` interface has no parameter for the `ukevent` structure. Instead, the user-space process is expected to place one or more `ukevent` structures immediately after the `kevent_user_control` structure in memory, and to set the `num` field to how many of those structures are present. A process interested in events should create this set of structures and pass them to `kevent_ctl()` with a `cmd` value of `KEVENT_CTL_ADD`. After that, the kernel will start generating events at the appropriate times. Other possible `cmd` values are `KEVENT_CTL_REMOVE` and `KEVENT_CTL_MODIFY`, which have the obvious effect. 

The final supported command is `KEVENT_CTL_WAIT`, which will wait for the number of events specified in the `num` field. An optional timeout value can also be provided. The returned events will, once again, go into memory just after the `kevent_user_control` structure. It is also possible to pass the kevent file descriptor to `poll()` or `select()`. 

Extending this mechanism to asynchronous network I/O requires the addition of two more system calls: 

```
long aio_send(int kevent_fd, int socket_fd, void *buffer, size_t size,
                      unsigned flags);
        long aio_recv(int kevent_fd, int socket_fd, void *buffer, size_t size,
                      unsigned flags);
```

Either one of these calls will put together and enqueue a special kevent request on the given `kevent_fd` file descriptor. The I/O will remain outstanding; once it completes, the associated event will be returned to the process. Until the completion event, the `buffer` should not be touched. There is also a provision for an `aio_sendfile()` system call, though it has not yet been implemented. 

At the lower levels, enabling asynchronous I/O for a protocol requires the addition of two new methods to the `proto` structure: 

```
int	(*async_recv) (struct sock *sk, void *dst, size_t size);
        int (*async_send) (struct sock *sk, struct page **pages, 
                           unsigned int poffset, size_t size);
```

In Evgeniy's patch, only the TCP protocol has been extended in this manner. 

There has been very little discussion of this patch on the netdev mailing list (where it was posted). Your editor suspects that, while the functionality provided by the patch is welcome, the user-space interface, perhaps, needs a little bit of work before it will be ready for inclusion into the mainline kernel.
