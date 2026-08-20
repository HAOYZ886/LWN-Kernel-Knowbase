---
title: "UIO: user-space drivers"
url: https://lwn.net/Articles/232575/
date: "May 2, 2007"
category: "Device drivers-In user space"
author: 
---

The concept of supporting user-space drivers has appeared on this page a few times before. It's back; this time there is a version of the patch (now called "UIO") which is being proposed for inclusion into 2.6.22. The interface has changed somewhat, so another look is called for. 

Like the previous version, UIO does not completely eliminate the need for kernel-space code. A small module is required to set up the device, perhaps interface to the PCI bus, and register an interrupt handler. The last function (interrupt handling) is particularly important; much can be done in user space, but there needs to be an in-kernel interrupt handler which knows how to tell the device to stop crying for attention. 

The kernel module includes `<linux/uio_driver.h>`. If it's a driver for a PCI device, it should register itself as a PCI driver in the usual way. When it comes time to connect a device (perhaps in the PCI `probe()` function), the driver fills in a `uio_info` structure: 

```
struct uio_info {
    	char			*name;
    	char			*version;
    	struct uio_mem		mem[MAX_UIO_MAPS];
    	long			irq;
    	unsigned long		irq_flags;
    	void			*priv;
    	irqreturn_t (*handler)(int irq, struct uio_info *dev_info);
    	int (*mmap)(struct uio_info *info, struct vm_area_struct *vma);
    	int (*open)(struct uio_info *info, struct inode *inode);
    	int (*release)(struct uio_info *info, struct inode *inode);
    	/* Internal stuff omitted */
        };
```

Here, `name` is the name of the device and `version` is the driver version (which will show up in sysfs). The number of the interrupt used by the device (if any) goes into `irq`, with `irq_flags` being the flags which will be passed to `request_irq()`. The function which handles interrupts is `handler()`. This handler should acknowledge the interrupt; it usually does not need to do anything else. The `mmap()`, `open()`, and `release()` functions are called from the equivalent `file_operations` members. 

The `mem` array describes any memory areas which can be mapped into user space. The `uio_mem` structure looks like: 

```
struct uio_mem {
    	unsigned long addr;
    	unsigned long size;
    	int memtype;
    	void __iomem *internal_addr;
    	/* ... */
        };
```

For each mappable area, `addr` is the relevant address, and `size` is the size of the area. If it's an I/O memory area, `internal_addr` is the address returned by `ioremap()`. The `memtype` field describes what the area really is: 

  * `UIO_MEM_PHYS` indicates that `addr` is a physical address, generally for an I/O memory area. 

  * `UIO_MEM_LOGICAL` is memory in the kernel logical address space, such as that returned by `kmalloc()`. 

  * `UIO_MEM_VIRTUAL` is memory in the kernel virtual address space \- the space used by `vmalloc_user()` and friends. 

Once the structure is filled in, the driver stub passes it to: 

```
int uio_register_device(struct device *parent, struct uio_info *info);
```

The `parent` pointer tells the kernel which "real" device is associated with the UIO device; if the driver is for a PCI device, `parent` will be `pci_dev->dev`. 

There is not much more to the kernel-space UIO API. When a device goes away, the driver should call: 

```
void uio_unregister_device(struct uio_info *info);
```

The final function of note is: 

```
void uio_event_notify(struct uio_info *info);
```

Its purpose is to notify the UIO core that an event (typically an interrupt) has occurred. The stub driver need not call `uio_event_notify()` for real interrupts, but it can be used to simulate interrupts in other situations. 

On the user space side, the first UIO-handled device will show up as `/dev/uio0` (assuming a normal `udev` setup). The user-space driver will open the device. Reading the device returns an `int` value which is the event count (number of interrupts) seen by the device; if no interrupts have come in since the last read, the operation will block until an interrupt happens (though non-blocking operation is supported in the usual way as well). The file descriptor can be passed to `poll()`. 

The memory areas described by the kernel-space driver can be mapped into user space with the `mmap()` call. The interface is just a little strange: the `offset` value passed to `mmap()` should be _N_ times the page size for the _N_ th memory area. So, on a system with 4096-byte pages, the first memory area will be found with an offset of zero, the second at 4096, the third at 8192, etc. Once that is figured out, though, everything is pretty straightforward. 

There are some limitations, of course. UIO drivers are char drivers; there is no provision for creating user-space block or network drivers at this time. It is not possible to set up DMA operations from user space. But, for drivers which can be implemented with I/O memory access and simple interrupt handlers, the necessary pieces are in place. The patch set includes an example driver to show how it all works. [According to Thomas Gleixner](<https://lwn.net/Articles/232582/>), the original, fully in-kernel version of the driver had to implement 68 different `ioctl()` commands and was over 5,000 lines long. The associated user-space code was over 3,000 lines as well. The new driver eliminates all of that, with a total of 156 lines of kernel code and just under 3,000 lines in user space. 

Andrew Morton has [expressed some reservations](<https://lwn.net/Articles/232583/>) about the patch: 

I'm a bit uncertain about the whole UIO idea, really. I have this vague feeling that we'd prefer to encourage people to move device drivers into GPL'ed kernel rather than encouraging them to do closed-source userspace implementations which will probably end up being slower, less reliable and unavailable on various architectures, distros, etc 

The authors respond that it's not really about doing proprietary drivers, though some of that will undoubtedly go on. There's a number of people, especially in the embedded space, who want to do user-space drivers, for prototyping purposes if nothing else. The UIO framework gives them a relatively safe and standard way to write these drivers, which is seen as being better than having them each create their own kernel hooks. The patch has not been merged as of this writing, but, unless stronger objections arise, it's chances of getting into 2.6.22 are reasonably good.
