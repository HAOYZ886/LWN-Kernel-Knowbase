---
title: Ensuring data reaches disk
url: https://lwn.net/Articles/457667/
date: "September 7, 2011"
category: Data integrity
author: "September 7, 2011 This article was contributed by Jeff Moyer"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

September 7, 2011

This article was contributed by Jeff Moyer

In a perfect world, there would be no operating system crashes, power outages or disk failures, and programmers wouldn't have to worry about coding for these corner cases. Unfortunately, these failures are more common than one would expect. The purpose of this document is to describe the path data takes from the application down to the storage, concentrating on places where data is buffered, and to then provide best practices for ensuring data is committed to stable storage so it is not lost along the way in the case of an adverse event. The main focus is on the C programming language, though the system calls mentioned should translate fairly easily to most other languages. 

#### I/O buffering

In order to program for data integrity, it is crucial to have an understanding of the overall system architecture. Data can travel through several layers before it finally reaches stable storage, as seen below: 

> ![\[Data
flow diagram\]](https://static.lwn.net/images/2011/jm-data-flow.png)

At the top is the running application which has data that it needs to save to stable storage. That data starts out as one or more blocks of memory, or buffers, in the application itself. Those buffers can also be handed to a library, which may perform its own buffering. Regardless of whether data is buffered in application buffers or by a library, the data lives in the application's address space. The next layer that the data goes through is the kernel, which keeps its own version of a write-back cache called the page cache. Dirty pages can live in the page cache for an indeterminate amount of time, depending on overall system load and I/O patterns. When dirty data is finally evicted from the kernel's page cache, it is written to a storage device (such as a hard disk). The storage device may further buffer the data in a volatile write-back cache. If power is lost while data is in this cache, the data will be lost. Finally, at the very bottom of the stack is the non-volatile storage. When the data hits this layer, it is considered to be "safe." 

To further illustrate the layers of buffering, consider an application that listens on a network socket for connections and writes data received from each client to a file. Before closing the connection, the server ensures the received data was written to stable storage, and sends an acknowledgment of such to the client. 

After accepting a connection from a client, the application will need to read data from the network socket into a buffer. The following function reads the specified amount of data from the network socket and writes it out to a file. The caller already determined from the client how much data is expected, and opened a file stream to write the data to. The (somewhat simplified) function below is expected to save the data read from the network socket to disk before returning. 

> 

```
>      0 int
>      1 sock_read(int sockfd, FILE *outfp, size_t nrbytes)
>      2 {
>      3      int ret;
>      4      size_t written = 0;
>      5      char *buf = malloc(MY_BUF_SIZE);
>      6 
>      7      if (!buf)
>      8              return -1;
>      9
>     10      while (written < nrbytes) {
>     11              ret = read(sockfd, buf, MY_BUF_SIZE);
>     12              if (ret =< 0) {
>     13                      if (errno == EINTR)
>     14                              continue;
>     15                      return ret;
>     16              }
>     17              written += ret;
>     18              ret = fwrite((void *)buf, ret, 1, outfp);
>     19              if (ret != 1)
>     20                      return ferror(outfp);
>     21      }
>     22 
>     23      ret = fflush(outfp);
>     24      if (ret != 0)
>     25              return -1;
>     26 
>     27      ret = fsync(fileno(outfp));
>     28      if (ret < 0)
>     29              return -1;
>     30      return 0;
>     31 }
>
```

Line 5 is an example of an application buffer; the data read from the socket is put into this buffer. Now, since the amount of data transferred is already known, and given the nature of network communications (they can be bursty and/or slow), we've decided to use libc's stream functions (fwrite() and fflush(), represented by "Library Buffers" in the figure above) in order to further buffer the data. Lines 10-21 take care of reading the data from the socket and writing it to the file stream. At line 22, all data has been written to the file stream. On line 23, the file stream is flushed, causing the data to move into the "Kernel Buffers" layer. Then, on line 27, the data is saved to the "Stable Storage" layer shown above. 

#### I/O APIs

Now that we've hopefully solidified the relationship between APIs and the layering model, let's explore the intricacies of the interfaces in a little more detail. For the sake of this discussion, we'll break I/O down into three different categories: system I/O, stream I/O, and memory mapped (mmap) I/O. 

System I/O can be defined as any operation that writes data into the storage layers accessible only to the kernel's address space via the kernel's system call interface. The following routines (not comprehensive; the focus is on write operations here) are part of the system (call) interface: 

> Operation| Function(s)  
> ---|---  
> Open| `open()`, `creat()`  
> Write| `write()`, `aio_write()`, `pwrite()`, `pwritev()`  
> Sync| `fsync()`, `sync()`  
> Close| `close()`  
  
Stream I/O is I/O initiated using the C library's stream interface. Writes using these functions may not result in system calls, meaning that the data still lives in buffers in the application's address space after making such a function call. The following library routines (not comprehensive) are part of the stream interface: 

> Operation| Function(s)  
> ---|---  
> Open| `fopen()`, `fdopen()`, `freopen()`  
> Write| `fwrite()`, `fputc()`, `fputs()`, `putc()`, `putchar()`, `puts()`  
> Sync| `fflush()`, followed by `fsync()` or `sync()`  
> Close| `fclose()`  
  
Memory mapped files are similar to the system I/O case above. Files are still opened and closed using the same interfaces, but access to the file data is performed by mapping that data into the process' address space, and then performing memory read and write operations as you would with any other application buffer. 

> Operation| Function(s)  
> ---|---  
> Open| `open()`, `creat()`  
> Map| `mmap()`  
> Write| `memcpy()`, `memmove()`, `read()`, or any other routine that writes to application memory   
> Sync| `msync()`  
> Unmap| `munmap()`  
> Close| `close()`  
  
There are two flags that can be specified when opening a file to change its caching behavior: `O_SYNC` (and related `O_DSYNC`), and `O_DIRECT`. I/O operations performed against files opened with `O_DIRECT` bypass the kernel's page cache, writing directly to the storage. Recall that the storage may itself store the data in a write-back cache, so `fsync()` is still required for files opened with `O_DIRECT` in order to save the data to stable storage. The `O_DIRECT` flag is only relevant for the system I/O API. 

Raw devices (`/dev/raw/raw` _N_) are a special case of `O_DIRECT` I/O. These devices can be opened without specifying `O_DIRECT`, but still provide direct I/O semantics. As such, all of the same rules apply to raw devices that apply to files (or devices) opened with `O_DIRECT`. 

Synchronous I/O is any I/O (system I/O with or without `O_DIRECT`, or stream I/O) performed to a file descriptor that was opened using the `O_SYNC` or `O_DSYNC` flags. These are the synchronous modes, as defined by POSIX: 

  * `O_SYNC`: File data and all file metadata are written synchronously to disk. 

  * `O_DSYNC`: Only file data and metadata needed to access the file data are written synchronously to disk. 

  * `O_RSYNC`: Not implemented 

The data and associated metadata for write calls to such file descriptors end up immediately on stable storage. Note the careful wording, there. Metadata that is not required for retrieving the data of the file may not be written immediately. That metadata may include the file's access time, creation time, and/or modification time. 

It is also worth pointing out the subtleties of opening a file descriptor with `O_SYNC` or `O_DSYNC`, and then associating that file descriptor with a libc file stream. Remember that `fwrite()`s to the file pointer are buffered by the C library. It is not until an `fflush()` call is issued that the data is known to be written to disk. In essence, associating a file stream with a synchronous file descriptor means that an `fsync()` call is not needed on the file descriptor after the `fflush()`. The `fflush()` call, however, is still necessary. 

#### When Should You Fsync?

There are some simple rules to follow to determine whether or not an `fsync()` call is necessary. First and foremost, you must answer the question: is it important that this data is saved now to stable storage? If it's scratch data, then you probably don't need to `fsync()`. If it's data that can be regenerated, it might not be that important to `fsync()` it. If, on the other hand, you're saving the result of a transaction, or updating a user's configuration file, you very likely want to get it right. In these cases, use `fsync()`. 

The more subtle usages deal with newly created files, or overwriting existing files. A newly created file may require an `fsync()` of not just the file itself, but also of the directory in which it was created (since this is where the file system looks to find your file). This behavior is actually file system (and mount option) dependent. You can either code specifically for each file system and mount option combination, or just perform `fsync()` calls on the directories to ensure that your code is portable. 

Similarly, if you encounter a system failure (such as power loss, `ENOSPC` or an I/O error) while overwriting a file, it can result in the loss of existing data. To avoid this problem, it is common practice (and advisable) to write the updated data to a temporary file, ensure that it is safe on stable storage, then rename the temporary file to the original file name (thus replacing the contents). This ensures an atomic update of the file, so that other readers get one copy of the data or another. The following steps are required to perform this type of update: 

  1. create a new temp file (on the same file system!) 
  2. write data to the temp file 
  3. `fsync()` the temp file 
  4. rename the temp file to the appropriate name 
  5. `fsync()` the containing directory 

#### Checking For Errors

When performing write I/O that is buffered by the library or the kernel, errors may not be reported at the time of the `write()` or the `fflush()` call, since the data may only be written to the page cache. Errors from writes are instead often reported during calls to `fsync()`, `msync()` or `close()`. Therefore, it is very important to check the return values of these calls. 

#### Write-Back Caches

This section provides some general information on disk caches, and the control of such caches by the operating system. The options discussed in this section should not affect how a program is constructed at all, and so this discussion is intended for informational purposes only. 

The write-back cache on a storage device can come in many different flavors. There is the volatile write-back cache, which we've been assuming throughout this document. Such a cache is lost upon power failure. However, most storage devices can be configured to run in either a cache-less mode, or in a write-through caching mode. Each of these modes will not return success for a write request until the request is on stable storage. External storage arrays often have a non-volatile, or battery-backed write-cache. This configuration also will persist data in the event of power loss. From an application programmer's point of view, there is no visibility into these parameters, however. It is best to assume a volatile cache, and program defensively. In cases where the data is saved, the operating system will perform whatever optimizations it can to maintain the highest performance possible. 

Some file systems provide mount options to control cache flushing behavior. For ext3, ext4, xfs and btrfs as of kernel version 2.6.35, the mount option is "`-o barrier`" to turn barriers (write-back cache flushes) on (the default), or "`-o nobarrier`" to turn barriers off. Previous versions of the kernel may require different options ("`-o barrier=0,1`"), depending on the file system. Again, the application writer should not need to take these options into account. When barriers are disabled for a file system, it means that fsync calls will not result in the flushing of disk caches. It is expected that the administrator knows that the cache flushes are not required before she specifies this mount option. 

#### Appendix: some examples

This section provides example code for common tasks that application programmers often need to perform. 

  1. [Synchronizing I/O to a file stream](<https://lwn.net/Articles/457670/>)

  2. [Synchronizing I/O using file descriptors](<https://lwn.net/Articles/457671/>) (system I/O) This is actually a subset of the first example and is independent of the `O_DIRECT` open flag (so will work whether or not that flag was specified). 

  3. [Replacing an existing file](<https://lwn.net/Articles/457672/>) (overwrite). 

  4. [sync-samples.h](<https://lwn.net/Articles/459139/>) (needed by the above examples).
