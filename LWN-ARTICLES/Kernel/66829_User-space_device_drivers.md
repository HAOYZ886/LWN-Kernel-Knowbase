---
title: "User-space device drivers"
url: https://lwn.net/Articles/66829/
date: "January 20, 2004"
category: "Device drivers-In user space"
author: 
---

Peter Chubb works with the [Gelato](<http://gelato.unsw.edu.au/>) project, which works toward better Linux performance on the IA-64 architecture. Among other things, Peter is responsible for the 64-bit sector support which went into the 2.5 kernel. At Linux.Conf.Au, Peter discussed device drivers. He pointed out that drivers, while making up roughly 50% of the code in the kernel, are responsible for 85% of all kernel bugs. Drivers tend to be written by people who would not normally be considered kernel hackers: hardware engineers, for example. These people tend to have a hard time dealing with the special nature of kernel programming, where interfaces are fluid, bugs are lethal, and many normal development tools are not available. 

Driver authors - and their users - might have a much easier time if drivers could be written to run in user space. In addition to mitigating the above-mentioned kernel programming issues, user-space driver development would allow the creation of a stable ABI; it also, presumably, would eliminate any licensing issues associated with closed-source drivers. User-space driver writers could also use any language they choose, "even Python." 

Peter and company have set out to make user-space drivers possible. Some of the necessary pieces are already in place. Standard Linux will allow a suitably privileged process to access I/O ports, for example. Low-address memory-mapped I/O registers can be accessed via a `mmap()` of `/dev/mem`. There is also an interface which gives user-space processes access to the PCI configuration space; this interface works via `ioctl()` calls on `/proc` files, though, thus upsetting the sensibilities of most kernel hackers. These facilities are enough to allow some user-space drivers (particularly XFree86) to work, but they are not sufficient to enable a wider range of drivers to move out of the kernel. 

One of the big gaps is interrupts; there is no way, currently, for user-space processes to register and respond to device interrupts. A patch from the Gelato project addresses this gap by creating a set of files under `/proc`. A process wanting to deal with interrupt 11, say, would open `/proc/irq/11/irq`. Reading the resulting file descriptor enables the interrupt and blocks the process until a device interrupt happens; control then returns to user-space, which can figure out what to do. A typical user-space driver will set up a separate thread to wait for interrupts in this manner; the actual work can be handed off to a different thread within the program. 

Peter presented some graphs showing that interrupt response times suffer very little when interrupt handlers run in user space. The main limitation at the moment seems to be the fact that shared interrupts are not supported. 

Another thing that user-space processes cannot normally do is set up DMA operations. To enable DMA, a new set of system calls has been added. The interface appears to be in a bit of flux, but it will be something like the following. The driver starts by opening a special file for device operations: 

```
int usr_pci_open(int bus, int slot, int function);
```

There is then a function for setting up DMA mappings: 

```
int usr_pci_map(int fd, int cmd, struct mapping_info *info);
```

The `cmd` argument can be `USR_ALLOC_CONSISTENT` to set up a long-lived consistent mapping, or `USR_MAP` to create a streaming, scatter/gather mapping. In either case, the `info` argument is used to pass in the relevant information, and to get the necessary address(es). There is also, of course, a `USR_UNMAP` operation for when the DMA is complete. 

Many user-space drivers will be able to obtain their requests directly from user space; the X server works in this way. Many other drivers, however, will need to hook into the kernel for this information. The current patch includes a mechanism (Peter described it as ugly) for a user-space block driver to register itself with the kernel and get I/O requests. It works by opening another special file and using it to communicate requests and responses back and forth. A similar interface apparently exists for network drivers. 

Getting a user-space driver patch into the kernel could be an interesting challenge. Many kernel hackers, certainly, resist changes that look like they are pushing Linux toward something that looks like a microkernel architecture - or which might legitimize binary-only drivers. On the other hand, some drivers bring a great deal of baggage into the kernel with them which might be better kept in user space; think of some of the code required by some sound drivers or the modulation software needed by "linmodem" drivers. The ability to run these drivers in user space could be a nice thing to have. 

See [the Gelato user-level drivers page](<http://www.gelato.unsw.edu.au/IA64wiki/UserLevelDrivers>) for more information.
