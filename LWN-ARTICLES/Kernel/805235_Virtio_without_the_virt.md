---
title: "Virtio without the \"virt\""
url: https://lwn.net/Articles/805235/
date: "November 22, 2019"
category: "Virtualization-virtio"
author: "November 22, 2019 This article was contributed by Stefan Hajnoczi and Michael Tsirkin KVM Forum"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

November 22, 2019

This article was contributed by Stefan Hajnoczi and Michael Tsirkin

* * *

[KVM Forum](<https://lwn.net/Archives/ConferenceByYear/#2019-KVM_Forum>)

When [virtio](<https://www.linux-kvm.org/page/Virtio>) was merged in Linux v2.6.24, its author, Rusty Russell, [described the goal](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ec3d41c4db4c>) as being for ""common drivers to be efficiently used across most virtual I/O mechanisms"". Today, much progress has been made toward that goal, with virtio supported by multiple hypervisors and guest drivers shipped by many operating systems. But these applications of virtio are implemented in software, whereas Michael Tsirkin's "[VirtIO without the Virt](<https://kvmforum2019.sched.com/event/TmxF/virtio-without-the-virt-towards-implementations-in-hardware-michael-tsirkin-red-hat>)" talk at KVM Forum 2019 laid out how to implement virtio in hardware.

#### Motivation

One might ask why it makes sense to implement virtio devices in hardware. After all, they were originally designed for hypervisors and have been optimized for software rather than hardware implementation. Now that virtio support is widespread, the network effects allow hardware implementations to reuse the guest drivers and infrastructure. The [virtio 1.1 specification](<https://docs.oasis-open.org/virtio/virtio/v1.1/virtio-v1.1.html>) defines ten device types, among them a network interface, SCSI host bus adapter, and console. Implementing a standards-compliant device interface lets hardware implementers focus on delivering the best device instead of designing a new device interface and writing guest drivers from scratch. Moreover, existing guests will work with the device out of the box, and applications utilizing user-space drivers, such as the [DPDK](<https://www.dpdk.org/>) packet processing toolkit, do not need to be relinked with new drivers — this is especially helpful when static linking is utilized.

Implementing virtio in hardware also makes it easy to switch between hardware and software implementations. A software device can be substituted without changing guest drivers if the hardware device is acting up. Similarly, if the driver is acting up, it is possible to substitute a software device to make debugging the driver easier. It is possible to assign hardware devices to performance-critical guests while assigning software devices to the other guests; this decision can be changed in the future to balance resource needs. Finally, implementing virtio in hardware makes it possible to live-migrate virtual machines more easily. The destination host can have either software or hardware virtio devices.

#### Implementing virtio PCI devices

Virtio has a number of useful properties for hardware implementers. Many device types have optional features so that device implementers can choose a subset that meets their use case. These optional features are negotiated during device initialization for forward and backward compatibility. This means hardware devices will continue working with guest drivers even after new versions of the virtio specification become widespread. Old guest drivers will work with newer devices too.

[ ![\[Michael Tsirkin\]](https://static.lwn.net/images/2019/kvmf-tsirkin-sm.jpg) ](<https://lwn.net/Articles/805240/>)

Historically, virtio was performance-optimized for software implementations. They used guest physical addresses instead of PCI bus addresses that are translated by an IOMMU. Memory coherency was also assumed and DMA memory-ordering primitives were therefore unnecessary. In preparation for hardware virtio implementations, the `VIRTIO_F_ORDER_PLATFORM` and `VIRTIO_F_ACCESS_PLATFORM` feature bits were introduced in virtio 1.1. A device that advertises these feature bits requires a driver that uses bus addresses and DMA memory-ordering primitives.

At least three approaches exist for hardware virtio PCI devices: full offloading, [virtual data path acceleration](<https://www.redhat.com/en/blog/achieving-network-wirespeed-open-standard-manner-introducing-vdpa>) (vDPA), and vDPA partitioning. Full offloading passes the entire device or a PCI [SR-IOV](<https://en.wikipedia.org/wiki/Single-root_input/output_virtualization>) virtual function (VF), which is a sub-device available on PCI adapters designed for virtualization, to the guest. All device accesses are handled in hardware — both those related to device initialization and to the data-path device operation. In this setup, all software is completely vendor-independent.

By comparison, vDPA is a hybrid software/hardware approach where a vendor-specific driver intercepts control path (discovery and initialization) accesses from the virtio driver and handles them in software, while the data path is implemented in hardware in a way compliant with the virtio specification. Performance is still good since the data path is handled directly in hardware.

The final approach is vDPA partitioning based on fine-grained memory protection between guests, such as the PCI process address space ID (PASID), which allow multiple virtual address spaces for device accesses instead of just one. This allows flexible resource allocation because users can configure the host driver to pass resources to guests as they wish. PASID support is not yet widespread, so this approach has not been explored as much as the alternatives.

Hardware bugs in fully offloaded devices that are not fixable in a firmware update can be assigned new virtio feature bits. Workarounds can be added to generic virtio drivers when these feature bits are seen. Hardware vendors can make the device's feature bits programmable, for example via a firmware update, so that the device refuses to start if the driver does not support a workaround for a critical bug. Bugs in vDPA devices can be worked around in the vendor's driver.

#### Live migration

Users often wish to move a running guest to another host with minimal downtime. When hardware devices are passed through to the guest, this becomes challenging because saving and restoring device state is not yet widely implemented for hardware devices. The details of representing device state are not covered by the virtio 1.1 specification, so hardware implementers must tackle this issue themselves.

[QEMU](<https://qemu.org/>) can help with live-migration compatibility by locking down the virtio feature bits that were negotiated on the source host and enforcing them on the destination host. This way, live migration ensures the availability of features that the guest is using. If the hardware device on the destination does not support the feature bits currently enabled on the source host, live migration is not possible.

During live migration, it is necessary to track writes to guest RAM because RAM is migrated incrementally in the background while the guest continues to run on the old host for a period of time. If writes are missed, the destination host receives an outdated and incomplete copy of guest RAM. Hardware devices must participate in this process of logging writes. [Infrastructure for this](<https://lwn.net/Articles/805040/>) is expected to land in the VFIO [mediated device](<https://www.kernel.org/doc/html/latest/driver-api/vfio-mediated-device.html>) (mdev) driver subsystem in the future.

Looking further into the future, both vendor-independent support for live migration and the elimination of memory pinning should become possible as IOMMU capabilities grow. The new shared virtual addressing (SVA) support in Linux and associated IOMMU hardware allows devices to access a process address space instead of using a dedicated IOMMU page table. Using unpinned memory would be attractive because it enables swappable pages and memory overcommit. In addition, this will make write logging for live migration simpler because device writes into memory can cause faults and be tracked in a vendor-independent way.

PCI page request interface (PRI) is the mechanism that allows IOMMU fault handling, but it might not be sufficient to support post-copy live migration, where the guest immediately runs on the destination host without prior migration of guest RAM. In post-copy live migration, guest RAM is faulted in from the source host on demand with unpredictable latencies, something that might not be appropriate for PRI. Virtio might be able to help by standardizing a way for a device to request a page and to pause and resume request processing. The out-of-order properties of virtio queues mean that the device can proceed even as a specific request is blocked waiting for a page to be faulted.

#### Future optimizations

Finally, changes can be made to how virtio works to make hardware implementations faster. The amount of outstanding work available in a queue needs to be retrieved by the device from memory. Pushing this information to the device from the driver might help devices avoid memory accesses.

Today there exists an interrupt-suppression mechanism called "event index" that stores the associated state in guest RAM. Guest RAM accesses require hardware devices to perform DMA transfers, which can be expensive and waste PCI bus bandwidth if there have been no changes to RAM. A more hardware-friendly mechanism would be welcome here. In a similar vein, interrupt coalescing is a common technique to reduce CPU consumption due to interrupts being raised frequently. In hardware implementations it is easy to take advantage of this.

Participation in the [virtio Technical Committee](<https://www.oasis-open.org/committees/tc_home.php?wg_abbrev=virtio>) standardization process is easy and [open to anyone](<https://github.com/oasis-tcs/virtio-spec/#providing-feedback>). Hardware vendors are welcome to participate in order to improve support for their hardware.

#### Conclusion

The virtio specification was originally intended for software device implementations but is now being implemented in hardware devices as well. Tsirkin's presentation outlined how virtio 1.1 enables hardware implementations but also identified areas where further work is necessary, for example for live migration. Although hardware virtio devices are not common yet, the interest in hardware implementation from silicon vendors and cloud providers suggests the day is not far off when these "virtual" devices become physical.
