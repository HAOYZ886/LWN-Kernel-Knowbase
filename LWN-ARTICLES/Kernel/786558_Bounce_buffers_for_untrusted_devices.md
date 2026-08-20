---
title: Bounce buffers for untrusted devices
url: https://lwn.net/Articles/786558/
date: "April 26, 2019"
category: "Direct memory access; Security-Kernel hardening"
author: "April 26, 2019 This article was contributed by Marta Rybczyńska"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

April 26, 2019

This article was contributed by Marta Rybczyńska

The recently discovered [vulnerability in Thunderbolt](<https://lwn.net/Articles/782381/>) has restarted discussions about protecting the kernel against untrusted, hotpluggable hardware. That vulnerability, known as [Thunderclap](<http://thunderclap.io/>), allows a hostile external device to exploit [Input-Output Memory Management Unit (IOMMU)](<https://en.wikipedia.org/wiki/Input%E2%80%93output_memory_management_unit>) mapping limitations and access system memory it was not intended to. Thunderclap can be exploited by USB-C-connected devices; while we have seen USB attacks in the past, this vulnerability is different in that PCI devices, often considered as trusted, can be a source of attacks too. One way of stopping those attacks would be to make sure that the IOMMU is used correctly and restricts the device to accessing the memory that was allocated for it. Lu Baolu has [posted an implementation of that approach](<https://lwn.net/ml/linux-kernel/20190327063506.32564-1-baolu.lu%40linux.intel.com/>) in the form of bounce buffers for untrusted devices. 

#### PCI and untrusted devices

PCI devices are usually built into the system, there was not much concern about them going rogue (however, a reader [expressed concerns in the comments](<https://lwn.net/Articles/783485/>) on an LWN article about peer-to-peer PCI accesses). The PCI bus does support hotplugging, but its use is limited. It is, however, possible to attach external PCI devices to a bus like Thunderbolt. That opens the door to the Thunderclap vulnerability; a rogue device can benefit from the fact that the PCI bus is, in practice, more trusted than externally accessible buses.

The PCI bus does not have uncontrolled access to the system, though, on systems where an IOMMU exists and is in use. It allows (or denies) access by devices to specific memory regions and maps bus addresses to physical memory addresses. The IOMMU works at the page level, and the remapped regions must be set explicitly before use; each device has different regions it can access. However, not all systems have an IOMMU enabled (or even installed) because of performance concerns or functionality that does not work correctly with the IOMMU.

One step toward improving the situation is to keep track of which devices are expected to behave well and which might not. The marking of trusted and untrusted PCI devices [was added](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/drivers/acpi/property.c?h=v5.1-rc3&id=617654aae50eb59dd98aa53fb562e850937f4cde>) in December 2018. It is done with an `untrusted` flag added to [`struct pci_dev`](<https://elixir.bootlin.com/linux/v5.1-rc3/source/include/linux/pci.h#L287>) to control special handling of such devices, including full IOMMU mapping and functions like the bounce buffers. A PCI device is marked untrusted if the firmware marks its root port as external (currently only if the `ExternalFacingPort` ACPI property is set); that should be the case for Thunderbolt devices.

#### IOMMU constraints

Trusted PCI devices are expected to perform their DMA operations to and from the buffers they have been given to use; they do not run out of bounds or access other memory zones. With such devices, the IOMMU configuration code can take some shortcuts and, for example, map slightly bigger zones to fit hardware limitations and optimize IOMMU usage. For untrusted devices, we cannot make the same assumptions; the correct and strict configuration of the IOMMU becomes more important. Unfortunately, the minimum granularity of the (Intel) IOMMU is 4KB. Mapping a buffer with the IOMMU means allowing access to the whole 4KB page, even if the desired zone is smaller. 

One result of this limitation is that an unaware driver that allocates a small buffer for device DMA and maps it through the IOMMU exposes the whole page with all of the other data it may contain, even if it belongs to other drivers or to the kernel itself. The fact that this situation does not cause any runtime error could be considered a weak point of the DMA API. Just activating the IOMMU doesn't solve the problem -- the system must also take care to not place any unrelated data in the memory mapped by the IOMMU. 

#### Bounce buffers

This is where the proposed patch set comes into play. It implements bounce buffers for the untrusted devices; a bounce buffer is simply a separate memory area that is used for DMA operations. Data is copied ("bounced") between the original buffer and the bounce buffer, which is located in isolated memory that can be mapped by the IOMMU in such a way that there is no access to the data outside the buffer in question. 

If the original buffer covers a full page (or multiple full pages), nothing needs to change as this buffer can be directly mapped without exposing any unrelated data. If, instead, the buffer is inside a page that may contain other data, bounce buffers will be used. During the mapping, unmapping, and sync operations, the code will copy the data from the original buffer to the bounce buffer and back, depending on the direction of the transfer. Then the IOMMU uses the bounce-buffer addresses for the device instead of the original one.

When an I/O operation is set up, the original I/O buffer is split into three parts: "low", "middle", and "high". The low and high parts might lay on pages that may contain other data: they are the first and the last page that contains the device buffers. The middle pages contain only the device buffer, so they do not use the bounce buffer; only the low and high pages do. This operation may thus split a single contiguous buffer into three pieces; those pieces will be reunited (from the device's point of view) in the IOMMU mapping. 

The bounce-buffer patch implements another change: the IOMMU mapping is invalidated immediately after the unmap operation. If that mapping stays cached in the IOMMU, the device might still use it after the mapped page has been reallocated for some other purpose. The patch set also provides an option to deactivate the bounce buffers if the system administrator trusts the attached devices.

#### Similarity to swiotlb

In the discussion following [the first version of the patch set](<https://lwn.net/ml/linux-kernel/20190312060005.12189-1-baolu.lu@linux.intel.com/>), Christoph Hellwig [noted](<https://lwn.net/ml/linux-kernel/20190312163845.GA13141@infradead.org/>) that the code has similarities to the swiotlb (software input output translation lookaside buffer) subsystem. The swiotlb is a bounce-buffering mechanism used with devices that cannot access all of a system's memory. In response, Lu [tried to make use of the swiotlb code](<https://lwn.net/ml/linux-kernel/c1ffee0a-24d2-df53-0689-23b0d8ef065d@linux.intel.com/>), but that effort failed because the approach is somewhat different and the offsets given by the swiotlb are different than the the original ones for the low pages. This is because swiotlb copies the whole buffer, rather than just the low and high segments, during the mapping operation.

Robin Murphy [suggested](<https://lwn.net/ml/linux-kernel/04703ea4-58e5-f833-5b39-1b68df2839ca@arm.com/>) that the implementation should be made generic for the whole IOMMU subsystem and not limited to Intel VT-d only. The discussion continued after the second version submission and Lu proposed an [extension to swiotlb](<https://lwn.net/ml/linux-kernel/04b41eb3-5584-5c7d-5f5e-7c6f28a19b50@linux.intel.com/>). [A new version](<https://lwn.net/ml/linux-kernel/20190421011719.14909-1-baolu.lu@linux.intel.com/>) of the patch set was posted on April 21\. It includes a refactoring of the swiotlb and moves some of the driver-specific code to the generic IOMMU layer. 

#### Next steps

The use of bounce buffers can protect a system against a class of attacks. It remains an open question if there are more similar issues in the kernel and if there will be a need to harden other in-kernel interfaces. This is likely, as the threat model has completely changed -- the attacker now controls the devices that were previously thought of as trusted. It seems certain that we are going to see more attacks from rogue devices using unexpected protocols. The kernel interfaces that were considered internal in the past may need to be reviewed and hardened.

The implementation of the IOMMU bounce buffers is complete; one remaining question is what the performance penalty is. The measurements of the impact have not yet been submitted with the patch set. According to the description, the impact is expected to be small. One may expect that it should be lower than swiotlb since less data copying takes place. Large transfers should not be affected as they are usually page-aligned already. The overhead will be more visible for small transfers, where the setup will dominate the cost of a small copy.

The missing performance information, along with some other comments on the latest posting of the patch set, suggest that there is still some work to be done before this code is ready to go upstream. With luck, though, it shouldn't be too long before Linux systems have a higher level of protection against untrustworthy devices.
