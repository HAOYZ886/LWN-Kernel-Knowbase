---
title: Hotplug file descriptors and revoke()
url: https://lwn.net/Articles/328490/
date: "April 14, 2009"
category: Forced unmount; revoke
author: "By Jonathan Corbet April 14, 2009"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
April 14, 2009

Once upon a time, operating systems did not have to worry about hardware coming and going at awkward times. Whatever peripherals were bolted into the box when the system booted could be counted on to still be there at shutdown time. Contemporary systems don't work that way; devices will come and go at the whim of the user. Various subsystems have evolved mechanisms for coping with hardware which suddenly vanishes, but that code tends to be subsystem-specific and complex. Eric Biederman recently encountered this code and didn't really like what he saw. So he has set out to make something better. 

Eric's [patch series](<http://lwn.net/Articles/328267/>) begins with this observation: 

Not long after I touched the tun driver and made it safe to delete the network device while still holding it's file descriptor open I [saw] someone else touch the code adding a different feature and my careful work went up in flames. Which brought home another point: at the best of it this is ultimately complex tricky code that subsystems should not need to worry about. 

Eric also notes that the growth in hotplug-capable PCI devices will increase the number of subsystems and drivers which need to be prepared for this eventuality. Rather than spread hotplug-specific code through more parts of the kernel, he would like to create one central, well-supported mechanism. 

The issue that Eric is looking at in particular is: what happens to open file descriptors when the underlying resource goes away? Regardless of whether that resource is a physical device, a module, or something different altogether, the kernel needs to do a right thing when the file descriptor no longer points to something valid. Eric's patches create a new infrastructure which allows any subsystem to easily revoke access to a file descriptor in a more reliable and robust manner than has been seen before. 

The first issue that comes up is, invariably, `mmap()`. If a no-longer-existing device or file has been mapped into a process's address space, interesting and unpleasant things could happen. Eric's answer is a new function: 

```
void remap_file_mappings(struct file *file, 
        			     struct vm_operations_struct *vm_ops);
```

A call to `remap_file_mappings()` will locate every virtual memory area (VMA) associated with the given `file`. All mapped pages will be unmapped, making them inaccessible to the process which had mapped them. The operations associated with the VMA will be replaced with `vm_ops`; those operations will normally be `revoked_vm_ops`, which simply return a bus error whenever the process attempts to access one of the affected pages. 

The kernel also clearly needs to block any other operations - `read()`, `write()`, `ioctl()`, etc. - which might be performed on this file descriptor. The way to do that, of course, is to replace the `file_operations` structure associated with the file. The function to do that is: 

```
int fops_substitute(struct file *file, const struct file_operations *f_op,
    			struct vm_operations_struct *vm_ops);
```

One might imagine that this function could be quite simple, along the lines of: 

```
file->f_op = f_op;
        remap_file_mappings(file, vm_ops);
```

But the truth of the matter is rather more complicated. To begin with, there may be threads running in the old file operations, and some of those might be waiting for events which will, now, never happen. As a way of helping drivers unwedge themselves in this situation, Eric's patches add a new entry to `struct file_operations`: 

```
int (*awaken_all_waiters)(struct file *filp);
```

This function should cause any thread which is waiting for the given file to wake up and take note that the world has changed. 

The next sticking point is that, now that the file operations have been swapped out, there is no way for the underlying driver to know when all file descriptors have been closed. That is handled by waiting until there are no more known users of the old file operations, then calling the `release()` function directly from `fops_substitute()`. That leads to the sticky question of what happens if some thread never wakes up and the usage count never goes to zero; in the current patch, `fops_substitute()` will simply hang in this situation. 

Before one can even worry about that, though, there is the troublesome point that the kernel has no idea how many users of a given `file_operations` structure exist. So Eric has had to add a reference counting mechanism. In the new way of doing things, any kernel code must bracket calls into a file's `file_operations` with: 

```
int fops_read_lock(struct file *file);
        void fops_read_unlock(struct file *file, int revoked);
```

The return value from `fops_read_lock()` (which Eric invariably calls `fops_idx`) is non-zero if access to the file has already been revoked; it must be passed into the matching call to `fops_read_unlock()`. The biggest part of the patch series is a slog through the core VFS code adding locking around every `file_operations` access. That's a lot of little code changes which have to be made in a lot of places. 

There is a payoff, though: the handling of revoked files in various other subsystems can be ripped out and replaced with the new, generic code. The changes to the `/proc` filesystem, for example, leave the code almost 400 lines shorter. So the kernel gets smaller, and the new code, should, with luck, be more robust and more maintainable. 

This mechanism is useful for situations where devices disappear, but there is also a bigger goal in sight. There has long been a desire for a generic `revoke()` system call which would disconnect all open descriptors to a given file or device. It could be used to implement some sort of secure attention key, killing all processes which have open file descriptors to a console device, for example. `revoke()` would also be useful for forced unmounting of filesystems. It's a useful idea, with only one problem: `revoke()` is really hard. Nobody has yet come through with an implementation that looks complete and robust enough to be put into the kernel. 

Eric's patch set has not gotten there yet either. But it does represent another stab at the problem using an approach which, most developers agree, is the way that `revoke()` needs to be implemented. Over time, it might just evolve into the general solution which has evaded other developers for years.
