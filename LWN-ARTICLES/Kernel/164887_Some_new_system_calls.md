---
title: Some new system calls
url: https://lwn.net/Articles/164887/
date: "December 20, 2005"
category: System calls; preadv; splice; unshare
author: 
---

The addition of system calls to the kernel is a relatively rare event. Each new system call changes the interface presented to user space and creates an ABI which must be maintained forever. So new system calls are added only when there is a real need. That said, there is a fair variety of system call patches in circulation at the moment. 

**`mknodat()` and friends**

Ulrich Drepper, the maintainer of glibc, isn't just trying to add a system call; [his proposal](<http://lwn.net/Articles/164584/>) creates eleven of them. They are all variants on current file operations: 

```
int mknodat(int dfd, const char *pathname, mode_t mode, dev_t dev);
        int mkdirat(int dfd, const char *pathname, mode_t mode);
        int unlinkat(int dfd, const char *pathname);
        int symlinkat(const char *oldname, int newdfd, const char *newname);
        int linkat(int olddfd, const char *oldname, 
                   int newdfd, const char *newname);
        int renameat(int olddfd, const char *oldname,
                     int newdfd, const char *newname);
        int utimesat(int dfd, const char *filename, struct timeval *tvp);
        int chownat(int dfd, const char *path, uid_t owner, gid_t group);
        int openat(int dfd, const char *filename, int flags, int mode);
        int newfstatat(int dfd, char *filename, struct stat *buf, int flag);
        int readlinkat(int dfd, const char *pathname, char *buf, int size);
```

The pattern should be clear by now: each new system call extends an existing one by adding one or more "`dfd`" (default file descriptor) arguments. In each case, the new argument indicates a directory which is used instead of the current working directory when relative path names are provided. These calls can help applications work their way through directory trees in a race-free manner, and are also useful for implementing a virtual per-thread working directory. 

There was a minor comment on the implementation - Ulrich had wanted to avoid changing an exported function, but such changes are always fair game. Beyond that, there seems to be little resistance to adding these system calls. Expect them in a future kernel. 

**`pselect()` and `ppoll()`**

David Woodhouse, meanwhile, has been circulating [a patch](<https://lwn.net/Articles/164892/>) implementing the `pselect()` and `ppoll()` system calls. These calls each take a signal mask; that mask will be applied while the calling process waits for events, with the previous mask being restored on return. There is an emulated version of these calls in glibc now, but a truly robust implementation requires kernel support. As with most things involving signals, the new code gets somewhat complex in places. The end result, however, should be a pair of straightforward system calls which allow a process to apply a different signal mask while waiting for I/O. 

**`unshare()`**

The `unshare()` patch by Janak Desai was first covered here [last May](<http://lwn.net/Articles/135321/>). It allows a process to disconnect from resources which are shared with others. The target application is per-user namespaces; implementing these requires the ability to detach from the global namespace normally shared by all processes on the system. The [current version of this patch](<http://lwn.net/Articles/163772/>) implements namespace unsharing, but it also allows a process to privatize its view of virtual memory and open files. 

This patch has been through a fair amount of review, and has seen a number of improvements from that process. Andrew Morton's [reaction](<https://lwn.net/Articles/164907/>) to a request to include the patch in -mm suggests that there is some work yet to be done, though. Andrew wants to see a better justification for the patch; he is also concerned about the security implications of adding a relatively obscure bit of code. The end result is that Janak still has some homework to do before this patch will make it into the kernel. 

**`preadv()` and `pwritev()`**

The kernel currently supports the `pread()` and `pwrite()` system calls; these behave like `read()` and `write()`, with the exception that they take an explicit offset in the file. They will perform the operation at the given offset regardless of whether the "current" offset in the file has been changed by another thread, and they do not change the current offset as seen by any thread. Also supported are `readv()` and `writev()`, which perform scatter/gather I/O from the current file offset. The kernel does not have, however, any system call which combines these two modes of operation. 

It turns out that there are developers who wish they had system calls along the lines of: 

```
int preadv(unsigned int fd, struct iovec *vec, unsigned long vlen,
                   loff_t pos);
        int pwritev(unsigned int fd, struct iovec *vec, unsigned long vlen,
                    loff_t pos);
```

To satisfy this need, Badari Pulavarty has created a simple implementation which is currently part of the -mm tree. It seems that Ulrich Drepper suggested an alternative to adding two new system calls, however: change the `iovec` structure instead. Badari ran with that idea, posting [a new patch](<http://lwn.net/Articles/165036/>) creating a new `iovec` type: 

```
struct niovec
        {
            void __user *iov_base;
    	__kernel_size_t iov_len;
    	__kernel_loff_t iov_off; /* NEW */
        };
```

The new `iov_off` field is more flexible than plain `preadv()` in that it enables each segment in the I/O operation to have its own offset. The only down side is that the prototypes for the `readv()` and `writev()` methods in the `file_operations` structure must be changed. So every driver and filesystem which implements `readv()` and `writev()` breaks and must be changed. There are fewer of those than one might expect, but it is still a significant change. 

It was [suggested](<https://lwn.net/Articles/165196/>) that the asynchronous I/O operations could be used instead. The AIO interface already allows for the creation of vectored operations with per-segment offsets. The downside is that using AIO is more complicated in user space, heavier in the kernel, and, incidentally, AIO support in the kernel was never completed to the point where it will support these operations anyway. Still, that is an option which may need more consideration before changing one of the fundamental interfaces used by filesystems and drivers. 

**`splice()`**

Finally, there has been talk over many years of creating a `splice()` system call. The core idea is that a process could open a file descriptor for a data source, and another for a data sink. Then, with a call to `splice()`, those two streams could be connected to each other, and the data could flow from the source to the sink entirely within the kernel, with no need for user-space involvement and with minimal (or no) copying. 

Some of the infrastructure was put in place [one year ago](<http://lwn.net/Articles/119682/>) when Linus created a circular pipe buffer mechanism. Now Jens Axboe has put together [a simple `splice()` implementation](<http://lwn.net/Articles/164806/>) which uses that mechanism. The patch is not ready for prime time yet (Jens: ""I'm just posting this in the spirit of posting early""), but it is a beginning. In particular, it allows a file to be spliced to a pipe, as either the source or the sink. With a pair of splices, it is possible to set up an in-kernel file copy operation with no internal memory copying. 

Work left for the future includes cleaning up the ("ugly," "nasty") internal interfaces, and generalizing the code so that any two file descriptors can be spliced together. The ability to splice to network sockets would be particularly useful. Some of this may take a while, so don't expect `splice()` to show up in the mainline in the immediate future.
