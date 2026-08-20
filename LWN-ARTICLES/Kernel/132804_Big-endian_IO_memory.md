---
title: "Big-endian I/O memory"
url: https://lwn.net/Articles/132804/
date: "April 20, 2005"
category: IO memory
author: 
---

The kernel provides a set of functions for working easily with I/O memory. Those functions assume that the memory is stored in little-endian byte order. This assumption is usually valid - PCI peripherals, for example, are supposed to always use that ordering. There are devices out there, however, which export big-endian I/O memory. Dealing with these devices has required implementing special-purpose code in the drivers. 

One of the few significant changes merged after 2.6.12-rc2 is a new set of I/O memory functions for working with big-endian devices. These functions are: 

```
unsigned int ioread16be(void __iomem *addr);
        unsigned int ioread32be(void __iomem *addr)
        void iowrite16be (u16 datum, void __iomem *addr);
        viod iowrite32be (u32 datum, void __iomem *addr);
```

These functions will handle the necessary byte swapping (or lack thereof) to present properly-ordered values on the host architecture. They are exported to modules.
