---
title: A new I/O memory access mechanism
url: https://lwn.net/Articles/102232/
date: "September 15, 2004"
category: "  iomem; Device drivers-Support APIs; IO memory"
author: 
---

Most reasonably current cards for the PCI bus (and others) provide one or more I/O memory regions to the bus. By accessing those regions, the processor can communicate with the peripheral and make things happen. A look at `/proc/iomem` will show the I/O memory regions which have been registered on a given system. 

To work with an I/O memory region, a driver is supposed to map that region with a call to `ioremap()`. The return value from `ioremap()` is a magic cookie which can be passed to a set of accessor functions (with names like `readb()` or `writel()`) to actually move data to or from the I/O memory. On some architectures (notably x86), I/O memory is truly mapped into the kernel's memory space, so those accessor functions turn into a straightforward pointer dereference. Other architectures require more complicated operations. 

There have been some longstanding problems with this scheme. Drivers written for the x86 architecture have often been known to simply dereference I/O memory addresses directly, rather than using the accessor functions. That approach works on the x86, but breaks on other architectures. Other drivers, knowing that I/O memory addresses are not real pointers, store them in integer variables; that works until they encounter a system with a physical address space which doesn't fit into 32 bits. And, in any case, `readb()` and friends perform no type checking, and thus fail to catch errors which could be found at compile time. 

The 2.6.9 kernel will contain a series of changes designed to improve how the kernel works with I/O memory. The first of these is a new `__iomem` annotation used to mark pointers to I/O memory. These annotations work much like the `__user` markers, except that they reference a different address space. As with `__user`, the `__iomem` marker serves a documentation role in the kernel code; it is ignored by the compiler. When checking the code with `sparse`, however, developers will see a whole new set of warnings caused by code which mixes normal pointers with `__iomem` pointers, or which dereferences those pointers. 

The next step is the addition of a new set of accessor functions which explicitly require a pointer argument. These functions are: 

```
unsigned int ioread8(void __iomem *addr);
        unsigned int ioread16(void __iomem *addr);
        unsigned int ioread32(void __iomem *addr);
        void iowrite8(u8 value, void __iomem *addr);
        void iowrite16(u16 value, void __iomem *addr);
        void iowrite32(u32 value, void __iomem *addr);
```

By default, these functions are simply wrappers around `readb()` and friends. The explicit pointer type for the argument will generate warnings, however, if a driver passes in an integer type. 

There are "string" versions of these operations: 

```
extern void ioread8_rep(void __iomem *port, void *buf, 
                                unsigned long count);
```

All of the other variants are defined as well, of course. 

There is actually one other twist to these functions. Some drivers have to be able to use either I/O memory or I/O ports, depending on the architecture and the device. Some such drivers have gone to considerable lengths to try to avoid duplicating code in those two cases. With the new accessors, a driver which finds it needs to work with x86-style ports can call: 

```
void __iomem *ioport_map(unsigned long port, unsigned int count);
```

The return value will be a cookie which allows the mapped ports to be treated as if they were I/O memory; functions like `ioread8()` will automatically do the right thing. For PCI devices, there is a new function: 

```
void __iomem *pci_iomap(struct pci_dev *dev, int base, 
                                unsigned long maxlen);
```

For this function, the `base` can be either a port number or an I/O memory address, and the right thing will be done. 

As of 2.6.9-rc2, there are no in-tree users of the new interface. That can be expected to change soon as patches get merged and the kernel janitors get to work. For more information on the new I/O memory interface and the motivation behind it, see [this explanation from Linus](<https://lwn.net/Articles/102240/>).
