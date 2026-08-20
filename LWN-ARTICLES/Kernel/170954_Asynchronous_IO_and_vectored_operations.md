---
title: Asynchronous I/O and vectored operations
url: https://lwn.net/Articles/170954/
date: "February 7, 2006"
category: Asynchronous IO; Vectored IO
author: 
---

The `file_operations` structure contains pointers to the basic I/O operations exported by filesystems and char device drivers. This structure currently contains three different methods for performing a read operation: 

```
ssize_t (*read) (struct file *filp, char __user *buffer, size_t size, 
                         loff_t *pos);
        ssize_t (*readv) (struct file *filp, const struct iovec *iov, 
                          unsigned long niov, loff_t *pos);
        ssize_t (*aio_read) (struct kiocb *iocb, char __user *buffer, 
                             size_t size, loff_t pos);
```

Normal read operations end up with a call to the `read()` method, which reads a single segment from the source into the supplied buffer. The `readv()` method implements the system call by the same name; it will read one segment and scatter it into several user buffers, each of which is described by an `iovec` structure. Finally, `aio_read()` is invoked in response to asynchronous I/O requests; it reads a single segment into the supplied buffer, possibly returning before the operation is complete. There is a similar set of three methods for write operations. 

Back in November, Zach Brown posted [a vectored AIO patch](<http://lwn.net/Articles/158947/>) intended to provide a combination of the vectored (readv()/writev()) operations and asynchronous I/O. To that end, it defined a couple of new AIO operations for user space, and added two more `file_operations` methods: `aio_readv()` and `aio_writev()`. There was some resistance to the idea of creating yet another pair of operations, and a feeling that there was a better way. The result, after work by Christoph Hellwig and Badari Pulavarty, is [a new vectored AIO patch](<http://lwn.net/Articles/170752/>) with a much simpler interface - at the cost of a significant API change. 

The observation was made that a number of subsystems use vectored I/O operations internally in all cases, even in the case of a "scalar" `read()` or `write()` call. For example, the `read()` function in the current mainline pipe driver is: 

```
static ssize_t
        pipe_read(struct file *filp, char __user *buf, size_t count, loff_t *ppos)
        {
    	struct iovec iov = { .iov_base = buf, .iov_len = count };
    	return pipe_readv(filp, &iov, 1, ppos);
        }
```

Here, the `read()` method is essentially superfluous; it is provided simply because the API requires it. So, it was asked, rather than adding more vectored I/O operations, why not just "vectorize" the standard API? The resulting patch set brings about that change in a couple of steps. 

The first of those is to change the prototypes for the asynchronous I/O methods to: 

```
ssize_t (*aio_read) (struct kiocb *iocb, const struct iovec *iov, 
                 unsigned long niov, loff_t pos);
        ssize_t (*aio_write) (struct kiocb *iocb, const struct iovec *iov,  
                 unsigned long niov, loff_t pos);
```

Thus, the single buffer has been replaced with an array of `iovec` structures, each describing one segment of the I/O operation. For the current single-buffer AIO read and write commands, the new code creates a single-entry `iovec` array and passes it to the new methods. (It's worth noting that, as the code is currently written, that `iovec` array is no longer valid after `aio_read()` or `aio_write()` returns; that array will need to be copied for any operation which remains outstanding when those functions finish). 

The prototypes of a couple of VFS helper functions (`generic_file_aio_read()` and `generic_file_aio_write()`) have been changed in a similar manner. These changes ripple through every driver and filesystem providing AIO methods, making the patch reasonably large. A second patch then adds two new AIO operations (`IOCB_CMD_PREADV` and `IOCB_CMD_PWRITEV`) to the user-space interface, making vectored asynchronous I/O available to applications. 

The patch set then goes one step further by eliminating the `readv()` and `writev()` methods altogether. With this patch in place, any filesystem or driver which wishes to provide vectored I/O operations must do so via `aio_read()` and `aio_write()` instead. Note that this change does not imply that asynchronous operations themselves must be supported - it is entirely permissible (if suboptimal) for `aio_read()` and `aio_write()` to operate synchronously at all times. But this patch does make it necessary for modules wishing to provide vectored operations to, at a minimum, provide the `file_operations` methods for asynchronous I/O. If the AIO methods are not available for a given device or filesystem, a call to `readv()` or `writev()` will be emulated through multiple calls to `read()` or `write()`, as usual. 

Finally, with this patch in place, it is possible for a driver or filesystem to omit the `read()` and `write()` methods altogether if the asynchronous versions are provided. If, for example, only `aio_read()` is provided, all `read()` and `readv()` system calls will be handled by the `aio_read()` method. If, someday, all code implements the AIO methods, the regular `read()` and `write()` methods could be removed altogether. That would result in an interface which contained only one method for all read operations (and one more for writes). This change would also realize the vision expressed at the [2003 Kernel Summit](<http://lwn.net/Articles/40810/>) that all I/O paths inside the kernel would, in the end, be made asynchronous. 

There has been little discussion of the current patch set, so it is hard to predict what may ultimately become of it. Given that it simplifies a core kernel API while simultaneously making it more powerful, however, chances are that some version of this patch will find its way into the kernel eventually. 

(For more information on the AIO interface, see [this Driver Porting Series article](<http://lwn.net/Articles/24366/>) or chapter 15 of [LDD3](<http://lwn.net/Kernel/LDD3/>)).
