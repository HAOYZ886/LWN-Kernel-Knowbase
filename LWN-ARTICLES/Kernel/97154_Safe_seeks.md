---
title: Safe seeks
url: https://lwn.net/Articles/97154/
date: "August 10, 2004"
category: "Non-seekable objects; Seeking-Safely; llseek"
author: 
---

The `lseek()` system call allows user space to move the current read/write position within a file. It is not an operation which normally attracts attention, since its full effect is, normally, to change an internal integer index. It turns out, however, that `lseek()` has been poorly implemented in many parts of the kernel. The recent [vulnerability](</Vulnerabilities/96389/>) discovered by Paul Starzetz has highlighted the problem, with the result that the internal handling of `lseek()` is changing significantly for 2.6.8. 

Seeking within a file is straightforward; it is just a matter of changing the current position index inside the kernel. The situation gets a little murkier, however, when dealing with things that are not regular files. Virtual files implemented by the kernel can often be seeked in a meaningful way, if it's done carefully; the same is true of a very small number of physical devices. For most devices, however, along with objects like network connections, seeking makes no sense at all. 

The default behavior for `lseek()` is to change the internal offset pointer and return success; if code for the the underlying object (device, network connection, file, etc.) has not provided its own `llseek()` method, the call appears to succeed. Implementation of a non-seekable device requires an explicit action, instead, to ensure that user space is given the proper error. The traditional way of handling `lseek()` within a device driver is to include a simple `llseek()` method which looks like this: 

```
loff_t my_llseek(struct file *file, loff_t offset, int whence)
        {
            return -ESPIPE;    /* Not seekable */
        }
```

More recent kernels (2.4 and beyond) also provide a `no_llseek()` helper which looks like the above. 

This technique works, as long as the author bothers to do things this way. In some cases, this little step gets skipped, and the resulting object appears seekable even though it is not. Even when this method is provided, however, it is not a complete solution; the `pread()` and `pwrite()` system calls, which specify a specific offset for the operation, involve seeks. Objects within the kernel do not see these calls directly; they just look like regular `read()` and `write()` calls. This works because the internal methods for these calls are always passed the offset to use. 

What this means is that, for a non-seekable object, every `read()` or `write()` method should include a test like this: 

```
ssize_t my_read(struct file *filp, char *buf, size_t count,
        		    loff_t *ppos)
        {
        /* ... */
        if (ppos != &filp->f_pos)
            return -ESPIPE;
        /* ... */
        }
```

This test works because, for normal `read()` and `write()` calls, the `ppos` pointer goes directly to the offset (`f_pos`) stored in the `file` structure. If `ppos` points elsewhere, it means that a `pread()` or `pwrite()` call has been made, and an error should be returned. These tests are simple, but they are bits of boilerplate code which must be added to the implementation of all non-seekable objects, and not all authors bother. After all, for most uses, the code works just fine without. 

The above code also forces widespread knowledge of the contents of the `file` structure and how position information is passed to `read()` and `write()` methods. For `sysctl` methods, things are even worse: there is no position passed in, so there is no alternative to getting it from the `file` structure. 

Finally, there are some interesting race conditions associated with the handling of file offsets. Often a device driver will test a position for validity, sleep (while waiting for device operations or user-space copies), then change the offset. But that offset could have changed in other ways during the sleep, leaving its final value in an indeterminate state. 

In response to all this, Linus has thrown together a set of patches changing the way seeks are handled inside the kernel. These patches have found their way into 2.6.8-rc4, but they were not posted separately on any open mailing lists first. [The first patch](<https://lwn.net/Articles/97177/>) adds a new `FMODE_LSEEK` bit to the `file` structure, so that the virtual filesystem (VFS) code knows which files are seekable and which are not. The idea is to move all tests for illegal seeks to the core VFS code. [A second patch](<https://lwn.net/Articles/97178/>) adds separate mode bits for `pread()` and `pwrite()`; as it turns out, files implemented with the [seq_file interface](<https://lwn.net/Articles/22355/>) are seekable, but do not support those two calls. 

A pair of patches then followed to make use of the new tests in the VFS core. The [`nonseekable_open()`](<https://lwn.net/Articles/97179/>) helper was added to enable drivers (and other code) to clear the new bits and mark an object as not being seekable. It is meant to be called in the corresponding `open()` method. Then came [changes to a large number of drivers](<https://lwn.net/Articles/97180/>) making them use the new infrastructure; the net result was the removal of quite a bit of code. 

It's worth noting that this patch represents a change in how device drivers should be written, but the actual API has not been changed in any incompatible ways. Unmodified drivers will still work - at least, as well as they did before. [The sysctl change](<https://lwn.net/Articles/97181/>) does involve an API change, however. All `sysctl` methods now have the offset passed in explicitly as a parameter; they should no longer go digging through the `file` structure for that information. Unmodified `sysctl` implementations will no longer compile. 

[The final step](<https://lwn.net/Articles/97350/>) is to change how the `read()` and `write()` system calls are implemented. They now create a copy of the `f_pos` field and pass that to the appropriate methods, and copy the result back afterward. So those methods never work with `f_pos` directly, regardless of how they are invoked. As a result of all this work, the handling of seeking has become simpler and more robust.
