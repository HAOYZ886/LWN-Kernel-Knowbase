---
title: "ioctl(), the big kernel lock, and 32-bit compatibility"
url: https://lwn.net/Articles/115651/
date: "December 15, 2004"
category: Big kernel lock; ioctl
author: 
---

Despite efforts to remove the big kernel lock (BKL) from the 2.6 kernel, it still covers large amounts of code. Much of that code is implementations of the `ioctl()` method in device drivers and filesystems throughout the kernel. A poorly-implemented `ioctl()` method can block other processors for some time, wasting CPU time and creating high latencies. Fixing `ioctl()`'s BKL use has been on the "to do" list for some time, but nobody has dived in to get the job done. 

Mike Werner has recently taken a step in that direction, however, with [this patch](<https://lwn.net/Articles/114637/>) which aims to make it easy to wean driver `ioctl()` methods off the BKL one at a time. To that end, it creates a new method in the `file_operations` structure: 

```
int (*unlocked_ioctl) (struct inode *inode, struct file *file, 
                               unsigned int cmd, unsigned long arg);
```

This method behaves just like one would expect: if it is non-`NULL`, it will be called in preference to the regular `ioctl()` method, and the BKL will not be taken for that call. New drivers can be written to use this method, and the `ioctl()` methods of old drivers can be shifted over once they are known to be safe to call without the BKL. 

This is a different approach than was taken to get the BKL out of `lseek()` methods. In that case, the interface was changed by decree, and `lseek()` was called without the BKL. First, however, every in-tree `lseek()` method was enhanced with an explicit `lock_kernel()` call of its own. As a result, those methods still executed with the BKL held, but the taking of the BKL was made explicit and put into a place where it could be removed when it was no longer needed. A typical `ioctl()` method can be more complicated than most `lseek()` methods, however, so the creation of a new method must seem like the easier approach this time around. 

One commenter has suggested that the new method should not include the `inode` argument, since it is trivially obtained from the `file` structure anyway. The version of this patch which was merged into 2.6.10-rc3-mm1 retains that argument, however. 

Meanwhile, Michael Tsirkin has posted [a different `ioctl()` patch](<https://lwn.net/Articles/115751/>) which, while it provides a non-BKL migration path for that method, also solves another problem. One of the biggest challenges in writing portable `ioctl()` methods is dealing with 32-bit compatibility on 64-bit systems. When user space is running in 32-bit mode, it will have a different view of any structures passed into `ioctl()`, and the kernel must translate the 32-bit versions into something it can work with. 

The kernel provides some help with this translation in the form of a function called `register_ioctl32_conversion()`: 

```
typedef int (*ioctl_trans_handler_t)(unsigned int, unsigned int,
                                             unsigned long, struct file *);
        int register_ioctl32_conversion(unsigned int cmd, 
                                        ioctl_trans_handler_t handler)
```

After this call, any 32-bit `ioctl()` call using the given `cmd` will be passed to the `handler` function, which, presumably, knows how to deal with it. This mechanism works, but it has a few shortcomings. It relies on a global space for `ioctl()` command codes, for example. Every command is _supposed_ to be unique, but things do not always happen that way - especially with out-of-tree drivers. The use of a hash table to look up handler functions slows things down a bit. And, as [Andi Kleen pointed out recently](<https://lwn.net/Articles/115753/>), the current mechanism suffers from race conditions which appear to be unfixable without changing the interface. 

But, if you're going to change the interface, you might as well do it right. So Michael's patch adds two new `ioctl()` methods to the `file_operations` structure. The `ioctl_native()` method handles calls made from user-space processes which are using the same architecture model as the kernel, while `ioctl_compat()` is called in cases where the two differ. With this approach, the global table of commands can be eliminated, and its problems go away as well. Since the new `ioctl_compat()` method is invoked directly from the `file_operations` structure, it is easy to manage the module reference count to avoid unload races. 

Oh, and the kernel does not acquire the big kernel lock before calling either of the new methods; they are expected to be implemented with proper locking from the beginning. 

Michael's patch seems to solve all of the problems addressed by the `unlocked_ioctl()` approach, plus a few more. The debate has not yet begun, but it would not be surprising to see the two new methods win out in the end.
