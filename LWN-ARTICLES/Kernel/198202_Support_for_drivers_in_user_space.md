---
title: Support for drivers in user space
url: https://lwn.net/Articles/198202/
date: "September 4, 2006"
category: "Device drivers-In user space"
author: 
---

Device drivers are generally done inside the kernel for the usual reasons of performance and control. There are times, however, when the ability to run a device driver in user space is helpful. These include situations where the code is far too large to go into the kernel (X.org, for example) and where the author of the driver does not wish to place the code under the GPL. Some types of drivers (such as those for USB devices) are easily run in user space now, but others can be a bit more challenging. Very few PCI drivers, for example, are written in user space. 

Thomas Gleixner has written [an interface module](<http://lwn.net/Articles/197523/>) which may help to change that situation. With this code in place, PCI drivers (some of them, at least) can be written almost entirely in user space, with only a small stub module loaded into the kernel. 

That module has two specific jobs to carry out. The first is to register the device to be driven, with a couple of bits of important information. To that end, it should fill out an `iio_device` structure, which contains the following fields: 

```
struct iio_device {
        char			*name;
        char			*version;
        unsigned long		physaddr;
        void			*virtaddr;
        unsigned long		size;
        long			irq;
    
        irqreturn_t (*handler)(int irq, void *dev_id, struct pt_regs *regs);
        ssize_t (*event_write)(struct file *filep, const char __user * buf,
        		       size_t count, loff_t *ppos);
        struct file_operations	*fops;
        void			*priv;
        /* ... */
    };
```

The first part of the structure provides information about the hardware to be driven - its name, where its I/O memory area lives (`physaddr`), where that area has been mapped into the kernel (`virtaddr`), its size, and the interrupt being used by the device. If `virtaddr` is zero, then `physaddr` is interpreted as the beginning of a range of I/O ports, rather than a memory address. 

The `fops` field provides the file operations for the device; normally, they are set to the generic versions provided by the IIO (for "industrial I/O") driver: `iio_open()`, `iio_read()`, `iio_mmap()`, etc. With this setup, the driver can create a basic device which allows a user-space program to read from or write to device memory (or ports). I/O memory can also be mapped into user space. 

The capabilities described thus far are not all that different from what can be done with `/dev/mem`; the main difference is that the stub driver can enable the PCI device and perform any other needed initialization. The real hitch in writing user-space PCI drivers, however, has been in the handling of interrupts. There is currently no way to write a user-space interrupt handler, and the IIO patch doesn't really change that. Instead, the stub driver is expected to provide a minimal interrupt handler of its own. 

This handler is needed because every device requires its own specific interrupt acknowledgment ritual. The kernel must respond quickly to an interrupt and give the device the attention it craves so that said device will stop asserting the interrupt. After that, any additional processing can be done at relative leisure. So, once the handler provided with the stub driver acknowledges the interrupt, the rest of the work can normally be done by the user-space driver. 

All that is needed is to let this driver know that the interrupt has happened. The IIO module provides a couple of mechanisms for that purpose. One is a second device node associated with the device; whenever an interrupt happens, a byte can be read from this "event device." So a user-space driver can simply block on a read from that device, or it can use `poll()` in more complicated situations. It is also possible for the user-space driver to receive `SIGIO` signals when an interrupt happens, but using signals will normally increase the ultimate response time to the interrupt. 

So, to make all this happen, the stub driver provides a minimal interrupt handler in the `handler()` field of the `iio_device` structure. When an interrupt happens, the IIO module will call this handler; if it returns `IRQ_HANDLED`, user space will be notified. If the stub driver provides an `event_write()` function, that function will be called in response to a write operation on the event device. This capability can be used to further control the kernel-space response to interrupts, request that interrupts be masked, etc. 

Readers who think that the event mechanism shares some features with the proposed kevent subsystem are right. It is probable that the IIO event handling code will be rewritten to use kevents, if and when kevents are merged into the mainline. 

Meanwhile, however, the IIO driver works. Thomas has posted [an example driver](<https://lwn.net/Articles/198208/>) (or parts of one, anyway) to show how this mechanism can be used. The real question which appears to be on a number of minds, however, is: could ATI and nVidia use IIO to move their drivers out of the kernel. Only those vendors can answer that question, however, so, until they say something, nobody really knows.
