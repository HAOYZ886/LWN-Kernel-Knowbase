---
title: The new way of ioctl()
url: https://lwn.net/Articles/119652/
date: "January 18, 2005"
category: Big kernel lock; Device drivers; ioctl
author: 
---

The `ioctl()` system call has long been out of favor among the kernel developers, who see it as a completely uncontrolled entry point into the kernel. Given the vast number of applications which expect `ioctl()` to be present, however, it will not go away anytime soon. So it is worth the trouble to ensure that `ioctl()` calls are performed quickly and correctly - and that they do not unnecessarily impact the rest of the system. 

`ioctl()` is one of the remaining parts of the kernel which runs under the Big Kernel Lock (BKL). In the past, the usage of the BKL has made it possible for long-running `ioctl()` methods to create long latencies for unrelated processes. Recent changes, which have made BKL-covered code preemptible, have mitigated that problem somewhat. Even so, the desire to eventually get rid of the BKL altogether suggests that `ioctl()` should move out from under its protection. 

Simply removing the `lock_kernel()` call before calling `ioctl()` methods is not an option, however. Each one of those methods must first be audited to see what other locking may be necessary for it to run safely outside of the BKL. That is a huge job, one which would be hard to do in a single "flag day" operation. So a migration path must be provided. As of 2.6.11, that path will exist. 

The [patch ](<https://lwn.net/Articles/119656/>) (by Michael s. Tsirkin) adds a new member to the `file_operations` structure: 

```
long (*unlocked_ioctl) (struct file *filp, unsigned int cmd, 
                                unsigned long arg);
```

If a driver or filesystem provides an `unlocked_ioctl()` method, it will be called in preference to the older `ioctl()`. The differences are that the `inode` argument is not provided (it's available as `filp->f_dentry->d_inode`) and the BKL is not taken prior to the call. All new code should be written with its own locking, and should use `unlocked_ioctl()`. Old code should be converted as time allows. For code which must run on multiple kernels, there is a new `HAVE_UNLOCKED_IOCTL` macro which can be tested to see if the newer method is available or not. 

Michael's patch adds one other operation: 

```
long (*compat_ioctl) (struct file *filp, unsigned int cmd, 
                              unsigned long arg);
```

If this method exists, it will be called (without the BKL) whenever a 32-bit process calls `ioctl()` on a 64-bit system. It should then do whatever is required to convert the argument to native data types and carry out the request. If `compat_ioctl()` is not provided, the older conversion mechanism will be used, as before. The `HAVE_COMPAT_IOCTL` macro can be tested to see if this mechanism is available on any given kernel. 

The `compat_ioctl()` method will probably filter down into a few subsystems. Andi Kleen has posted patches adding new `compat_ioctl()` methods to the `block_device_operations` and `scsi_host_template` structures, for example, though those patches have not been merged as of this writing.
