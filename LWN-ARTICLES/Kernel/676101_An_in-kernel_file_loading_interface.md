---
title: "An in-kernel file loading interface"
url: https://lwn.net/Articles/676101/
date: "February 17, 2016"
category: "Filesystems-Virtual filesystem layer"
author: "By Jonathan Corbet February 17, 2016"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
February 17, 2016

One of the many interesting aspects to kernel development is that much of the kernel's functionality is, itself, not available to the kernel. Most system calls are not intended to be called internally. Traditionally, this feature gap has extended to the reading of files from the filesystem, an act which tends to look like the implementation of policy within the kernel and, potentially, opens up security issues; thus it has long been discouraged. 

Over time, though, we have seen the introduction of kernel code that does, indeed, read files. The first step in that direction was probably the [in-kernel module loader](<https://lwn.net/Articles/10921/>), which replaced the user-space loader back in 2002. The module loader does not actually open files; it depends on user space to hand it a file descriptor corresponding to the module to be loaded. But, given that, it does read the module code directly, perform the necessary symbol resolution, and bind it into the kernel. 

The door opened wider when the firmware-loading mechanism was moved in-kernel; in this case, the file containing the firmware is being opened by name from within the kernel. The [integrity management architecture](<https://lwn.net/Articles/227937/>) code also has to open files, and it seems likely that other uses will show up over time. Since there is no standard way to open and read a file within the kernel, there is a separate implementation for each of these users, each of which does things in its own way. 

Mimi Zohar recently decided that it was time to make file reading a first-class supported operation within the kernel; the result is [this patch set](<https://lwn.net/Articles/675665/>) adding a common file loader. It makes this operation easier to perform, but, as will be seen, it still seems like it's not really meant for common use. 

At the lowest level, Mimi's patch set adds a new function to read a file's contents into memory: 

```
int kernel_read_file(struct file *file, void **buf, loff_t *size,
    		         loff_t max_size, enum kernel_read_file_id id);
```

This function will read the data from the open file indicated by `file`; up to `max_size` bytes will be read. It will allocate a buffer (using `vmalloc()`) to hold the file's contents, storing a pointer in `*buf`; the caller should free the buffer when it is no longer needed. The actual length of the file will be placed in `*size`. If the file is larger than `max_size`, nothing will be allocated or read, and `-EFBIG` will be returned. 

The `id` argument is, arguably, where the interface (intentionally) loses a bit of generality. It is an `enum` type meant to indicate the purpose for which the file is being read; the values defined in the patch are `READING_KEXEC_IMAGE`, `READING_KEXEC_INITRAMFS`, `READING_FIRMWARE`, `READING_MODULE`, and `READING_POLICY`. The `READING_POLICY` option appears to be the motivation for the patch set; the IMA code can use it to read the policy and perform signature checking on the policy file. Developers wanting to use this interface will, most likely, have to add their own `kernel_read_file_id` constant to describe what they are doing. 

There are a couple of helpers built on top of `kernel_read_file()`: 

```
int kernel_read_file_from_path(char *path, void **buf, loff_t *size,
    			       	   loff_t max_size,
    				   enum kernel_read_file_id id);
        int kernel_read_file_from_fd(int fd, void **buf, loff_t *size,
        				 loff_t max_size, enum kernel_read_file_id id);
```

As might be expected, the first one opens and reads a file given its pathname, while the second takes an open file descriptor and reads from that. 

One advantage to implementing this functionality in a single place is that it becomes possible to apply a uniform security policy in all settings where the kernel tries to read a file. To that end, Mimi's patch set adds two new security hooks (`security_kernel_read_file()` and `security_kernel_post_read_file()`) that can pass judgment on file-reading operations. The `security_kernel_module_from_file()` and `security_kernel_fw_from_file()` hooks have been removed in favor of the new hooks. This is the purpose of the `kernel_file_read_id` parameter described above; it is passed to the loaded security module(s) and can be checked by the current security policy. 

This patch set has been through a few revisions and has gotten acknowledgments from a number of the relevant developers. At this point, there would appear to be few obstacles between it and the mainline kernel. So, in the near future, the kernel is likely to have a set of generic functions for opening and reading files, but any future users will have to tell the kernel what they are up to.
