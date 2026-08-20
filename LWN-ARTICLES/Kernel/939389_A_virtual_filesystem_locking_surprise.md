---
title: A virtual filesystem locking surprise
url: https://lwn.net/Articles/939389/
date: "July 31, 2023"
category: "Filesystems-Virtual filesystem layer"
author: "By Jonathan Corbet July 31, 2023"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
July 31, 2023

It is well understood that concurrency makes programming problems harder; the high level of concurrency inherent in kernel development is one of the reasons why kernel work can be challenging. Things can get even worse, though, if concurrent access happens in places where the code is not expecting it. The long story accompanying [this short patch](<https://lwn.net/ml/linux-fsdevel/20230724-vfs-fdget_pos-v1-1-a4abfd7103f3@kernel.org/>) from Christian Brauner is illustrative of the kind of problem that can arise when assumptions about concurrency prove to be incorrect. 

Within the kernel, [`struct file`](<https://elixir.bootlin.com/linux/v6.4.6/source/include/linux/fs.h#L959>) is used to represent an open file. It contains the information needed to work with that file, including an extensive [operations vector](<https://elixir.bootlin.com/linux/v6.4.6/source/include/linux/fs.h#L1771>), a reference count, a pointer to the associated inode, the current read/write position, and more. Since there can be multiple references to an open file, there must be a way to serialize access to this structure. The `f_lock` spinlock is used in most cases, but there is also a mutex called `f_pos_lock` that is used for access to the file position. 

Acquiring and releasing locks has a cost of its own. Many I/O operations affect the file position, so an I/O-intensive workload can end up repeatedly taking and releasing `f_pos_lock`, increasing the overhead imposed by the kernel. As it happens, though, having multiple references to an open file is a relatively rare occurrence. If there is only a single reference to a given file, concurrent access to the file position cannot happen and that lock overhead is wasted. To avoid this waste, the function that acquires `f_pos_lock` ([`__fdget_pos()`](<https://elixir.bootlin.com/linux/v6.4.6/source/fs/file.c#L1039>)) contains an optimization: 

```
if (file_count(file) > 1)
            mutex_lock(&file->f_pos_lock);
```

(The code has been simplified slightly to highlight the relevant part). The idea here is simple enough: if there is only a single reference to the file, concurrent access cannot happen and there is no point in taking the lock, so the `mutex_lock()` call is skipped. 

The [io_uring subsystem](<https://lwn.net/Articles/776703/>) has been under intensive development since its introduction in 2019; it is rapidly becoming an independent interface to much kernel functionality. There are currently efforts underway to add io_uring operations corresponding to [`waitid()`](<https://lwn.net/ml/linux-kernel/20230711204352.214086-1-axboe@kernel.dk/>), [futexes](<https://lwn.net/ml/linux-kernel/20230720221858.135240-1-axboe@kernel.dk/>), and [`getdents()`](<https://lwn.net/ml/linux-fsdevel/20230711114027.59945-1-hao.xu@linux.dev/>). That last patch, making the [`getdents()` system call](<https://man7.org/linux/man-pages/man2/getdents.2.html>) available in io_uring, is relevant here because `getdents()` relies heavily on the file position (and, possibly, state kept by the filesystem implementation) to allow a process to read through a long directory in multiple calls. 

The "fixed files" feature of io_uring is also relevant here; it lets a file be used numerous times in io_uring operations without the per-call overhead required with regular system calls. That overhead, which includes acquiring a reference to the file and validating the process's access to it, can be significant in I/O-heavy applications; fixing a file makes it possible to pay that cost only once, improving performance. When a file is fixed into io_uring, a new reference is created, so the reference count will increase. The process can, however, close its own file descriptor after fixing it in io_uring, leaving the fixed-file reference as the only one. The reference count will, as a result, drop back to one. It will also stay there while I/O operations on the file are underway in io_uring; the whole point of fixing the file is to avoid the cost of repeatedly gaining and releasing references. 

Brauner [pointed out](<https://lwn.net/ml/linux-fsdevel/20230712-alltag-abberufen-67a615152bee@brauner/>) a problem in the `getdents()` patch: if a file has been fixed in io_uring, and its reference count is one, it will be possible to run multiple `getdents()` operations concurrently within io_uring, each of which will access `f_pos` without taking the lock. The results of this concurrency are highly unlikely to be what the developer was hoping for. One might argue that this is a "then don't do that" sort of situation but, as Brauner described in his patch addressing the problem, io_uring is not the only way to run into trouble. 

In 2020, the kernel acquired an interesting system call named [`pidfd_getfd()`](<https://lwn.net/Articles/808997/>), which allows a suitably privileged process to extract an open file descriptor from a running process. This operation can be useful for, among other things, enabling a privileged supervisor process to perform operations that another process cannot perform on its own; opening a file outside of a container might be one example. For this to work, the file descriptor created by `pidfd_getfd()` must refer to the same open `file` structure as the descriptor in the target process. It creates a second reference to that structure, and the reference count is duly incremented to reflect that. 

A problem arises, though, if the target process has a `getdents()` call underway when its file descriptor is grabbed by `pidfd_getfd()`. Since, when `getdents()` was called, the file's reference count was one, the target process will not have acquired `f_pos_lock`. If the process that obtained the file descriptor with `pidfd_getfd()` also passes it to `getdents()`, things can go wrong. The second call will see the elevated reference count and acquire `f_pos_lock` but, since the first call did not acquire that lock, that acquisition will succeed immediately and the two `getdents()` call will run concurrently, once again with something other than the intended results. 

The fix is easy enough: simply remove the check on `f_count` and acquire `f_pos_lock` unconditionally. That will impose a performance cost, but nobody seems to have been worried enough about it to actually measure it. Linus Torvalds [applied the patch](<https://git.kernel.org/linus/20ea1e7d13c1b>) for the 6.5-rc4 release after editing the changelog (which he [described](<https://lwn.net/ml/linux-fsdevel/CAHk-=whfJhag+iEscftpVq=dHTeL7rQopCvH+Pcs8vJHCGNvXQ@mail.gmail.com/>) as ""*way* too much"", but which your editor found most useful). He also complained about how `pidfd_getfd()` shares the `file` structure, saying it would have been better to simply reopen the file (creating a new `file` structure); that would defeat the purpose for `pidfd_getfd()`, though, since the new file descriptor would no longer be usable to perform actions on the other process's behalf. 

Torvalds [remains grumpy](<https://lwn.net/ml/linux-fsdevel/CAHk-=wg8gY+oBoehMop2G8wq2L0ciApZEOOMpiPCL=6gxBgx=g@mail.gmail.com/>) about the shared access to `struct file` created by `pidfd_getfd()`, but it seems like it is here to stay. In any case, this problem has been fixed, clearing the way for the (eventual) use of `getdents()` on fixed files in io_uring. But it provides an example about how subtle assumptions regarding concurrency can go wrong in surprising ways.
