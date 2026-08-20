---
title: Preserving guest memory across kexec
url: https://lwn.net/Articles/895453/
date: "May 20, 2022"
category: Kexec; Virtualization; userfaultfd
author: "By Jonathan Corbet May 20, 2022 LSFMM"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
May 20, 2022

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2022/>)

The final session in the memory-management track at the [2022 Linux Storage, Filesystem, Memory-management and BPF Summit](<https://events.linuxfoundation.org/lsfmm/>) (LSFMM) was run remotely by James Gowans and David Woodhouse. It was titled "user-space control of memory mappings", with a subtitle of "letting guest memory and state survive kexec". Some options were discussed, but the real work is clearly yet to be done. 

The use case in question, Gowans began, is a live update of a hypervisor done with the kernel's [kexec](<https://en.wikipedia.org/wiki/Kexec>) functionality. To carry this out, the state of all running virtual machines is serialized to persistent storage, then kexec is used to boot into the updated hypervisor. After that, the virtual machines can all be restarted. The desire is to preserve the state of guest memory over the reboot, which means this memory cannot be managed by the host kernel in the traditional way; instead, the kernel should stay away from that memory and let user space manage its allocation to virtual machines. They have been looking at "sidecar virtual machines" as a way to implement this functionality. 

Most of guest memory, Gowans said, should not be touched by the new kernel, meaning that the kernel will only manage a small part of the memory given to guest systems. The [`userfaultfd()`](<https://man7.org/linux/man-pages/man2/userfaultfd.2.html>) system call is used to manage the rest; this is a change, since `userfaultfd()` only works with anonymous memory currently. Future requirements will include keeping I/O memory-management unit (IOMMU) mappings in sync, keeping DMA operations running while the update happens, and improving the speed of kexec by passing more state to the new kernel. 

John Hubbard asked if memory managed in this way needs to have associated `page` structures; the answer was that they are not needed. 

A few implementation options were presented. The first was a full filesystem, implemented in the kernel, that is used to manage allocations of reserved ranges of memory. The kernel would reconstruct this filesystem after a kexec. The [PKRAM mechanism](<https://lwn.net/ml/linux-kernel/1617140178-8773-1-git-send-email-anthony.yznaga@oracle.com/>), which preserves RAM contents over a kexec, would probably be used for this purpose; the PKRAM patches were posted last year, but have not been merged. How to handle other types of memory, such as PCI memory-mapped I/O (MMIO) registers, is an open question as well. 

The next implementation option was a FUSE-based filesystem; mapping of guest memory to page-frame numbers could then be handled from user space. A special control process could handle many of the details, and this solution would support mapping to PCI MMIO spaces. 

Finally, this feature could be implemented using a raw memory device, something along the lines of `/dev/mem`. The control process could use `ioctl()` calls to create and revoke mappings to pages in the guest process. User space would be charged with keeping mappings in place over the kexec call. There is evidently an implementation of this option running now. 

Jan Kara observed that there are a number of other things that need to be restored after a kexec, including open files and more. This task resembles [Checkpoint/Restore in User space](<https://criu.org/Main_Page>) (CRIU), which already exists. The response was that this solution does not try to recreate everything automatically; instead, hypervisor processes will be responsible for opening files again after the kexec. Woodhouse compared it to live migration to the same host. Gowans said that guests won't notice this happening; they will be paused and serialized, and their previous state pushed back into KVM by the new hypervisor. 

Returning to the implementation options, Gowans said that the full-filesystem approach offers the best latency and introspection, but it's not clear how MMIO regions can be handled. The FUSE approach gives full control to user space and solves the MMIO problem. The raw-memory version is the most flexible, but it requires reconstructing everything after the kexec, and is the least transparent to introspection. 

Next steps include figuring out how to handle IOMMU mappings, then picking an approach to pursue. The preferred approach looks like the FUSE version, so the plan is to put together an RFC patch implementing it and to have a polished version by the [KVM Forum](<https://events.linuxfoundation.org/kvm-forum/>) in September. 

Dan Williams said that the FUSE and raw-memory options look like the least scary ones. That said, PKRAM _does_ look scary; he asked about the status of those patches. David Hildenbrand answered that the last posting of that work "didn't inspire joy". 

The attendees were tired and the session wound down fairly quickly. The final question had to do with the existence of other use cases for this functionality. Hildenbrand suggested that databases could be a candidate. Specifically, huge, in-memory databases can take hours to boot and load up all of the data; a mechanism like this could possibly accelerate the process.
