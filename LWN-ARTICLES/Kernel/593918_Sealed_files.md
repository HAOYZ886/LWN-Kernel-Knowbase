---
title: Sealed files
url: https://lwn.net/Articles/593918/
date: "April 9, 2014"
category: "Filesystems-File sealing; Memfd"
author: "By Jonathan Corbet April 9, 2014"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
April 9, 2014

Interprocess communication using shared memory can be easy and efficient, but there is a catch: all of the processes involved must be able to trust each other. They must be able to assume that their peers will not modify memory contents after making them available; otherwise, no end of mischief is possible in the time between when memory contents are checked for sanity and when they are actually used. Similarly, processes need to trust that their peers will not truncate the file backing up a shared memory region at an inopportune time, causing fatal signals when they try to access parts of the region that no longer exist. In the real world, where this kind of trust is often not present, careful users of shared memory must copy data before using it and be prepared for signals; that kind of programming is cumbersome, error-prone, and slow. 

Developers have been talking about coming up with a solution to these problems for a bit; this discussion took the form of real code in mid-March, when David Herrmann posted his [file sealing patch set](<https://lwn.net/Articles/591108/>). The sealing concept allows one party in a shared-memory transaction to work with a memory segment in the knowledge that it cannot be changed by another process at an inopportune time. 

A process working with the sealing API will start by creating a file on an shmfs filesystem and mapping it into memory. That memory region is then filled with whatever data the process wants to pass to a another process on the system. When the segment is ready to be handed over, the process can seal it with a call to `fcntl()` using the new `SHMEM_SET_SEALS` command. There are three types of seal that can be set on a file: 

  * `SEAL_SHRINK` prevents the file from being reduced in size. 
  * `SEAL_GROW` disallows file growth. 
  * `SEAL_WRITE` prevents all modifications except resizing. 

If all three seals are set, then the file becomes immutable. Seals cannot be set on a file that has writable mappings; the creating process must remove all such mappings with `munmap()` before the `fcntl()` call. 

Once the file is sealed, the associated file descriptor can be passed to the peer process, which can verify that the seals are in place using the `SHMEM_GET_SEALS` `fcntl()` operation. If the seals are there, the recipient knows that the file (and associated shared memory region) cannot be changed in the indicated ways. That makes the use of zero-copy techniques much safer, and avoids a number of other potential issues as well. 

The actual enforcement of the seals is done within the shmfs filesystem. It is not hard to augment the calls implementing `write()` and `truncate()` to check for the existence of a seal and fail with `EPERM` should a seal exist. Since (as mentioned above) no writable mappings can exist when a seal is applied, all that is needed to prevent modification through memory mappings is a check in the `mmap()` implementation when a writable mapping is requested. It appears that the kernel can indeed credibly promise that a sealed file will not be changed in the indicated ways. 

One might argue that the potential for shared-memory mischief has just been replaced with the potential for seal-related attacks instead. The feature has been developed with an eye toward preventing such attacks, though. To start with, only shmfs supports sealing, so there should be no issues with hostile processes setting seals on real files. Once the initial seals have been set, they can only be changed by a process that has an _exclusive_ reference to the file. So a recipient process can verify that the seals are in place knowing that they cannot be removed as long as it holds its own reference to the file. So it should not be possible to perform denial-of-service attacks by placing seals on random files, and seals cannot be changed while another process is counting on their protection. 

For those who do not want to mount shmfs and work with files explicitly, there is also a new system call: 

```
int memfd_create(const char *name, u64 size, u64 flags);
```

This call will create a file (not visible in the filesystem) that is suitable for sealing; it will be associated with the given `name` and be `size` bytes in length. The return value on success will be a file descriptor associated with the newly created file. The only recognized flag is `MFD_CLOEXEC`, which maps to `O_CLOEXEC` internally, causing the file descriptor to be automatically closed if the process calls one of the forms of `exec()`. The returned file descriptor can be passed to `mmap()`, of course. 

Most commenters seemed happy enough with the proposed functionality, but there were a number of questions about the implementation and the semantics. Linus [didn't like the rules](<https://lwn.net/Articles/593957/>) regarding when seals could be changed; he suggested that, instead, only the creator of a file should be allowed to seal it. David [is not averse](<https://lwn.net/Articles/593959/>) to doing things that way; if sealing were made into a one-time operation, the reference counting on files could be eliminated entirely. He might also add a new flag (`MFD_ALLOW_SEALING`) to `memfd_create()` and restrict the sealing operations to files created with that flag set. 

Ted Ts'o, instead, [suggested](<https://lwn.net/Articles/593963/>) that sealing should not be limited to shmfs files. Instead, he would like to see consideration given to implementing this functionality in the virtual filesystem layer so that sealing could be used with files from any filesystem. David [responded](<https://lwn.net/Articles/593964/>) that he didn't see a use case for sealing in any other context, but Ted would still like to see a more general use for this functionality. This part of the conversation wound down without any resolution. 

There are a number of fairly immediate use cases for the sealing functionality in general. Graphics drivers could use it to safely receive buffers from applications. The upcoming [kdbus](<https://lwn.net/Articles/580194/>) transport can benefit from sealing. The Android "ashmem" allocator also implements a similar feature that could be moved over once this code gets upstream. So the chances of this functionality being merged into the mainline are fairly good, even though the details of how things will work have not yet been sealed in place.
