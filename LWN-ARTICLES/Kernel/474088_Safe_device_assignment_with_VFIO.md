---
title: Safe device assignment with VFIO
url: https://lwn.net/Articles/474088/
date: "January 3, 2012"
category: "Device drivers-In user space; Virtualization"
author: "By Jonathan Corbet January 3, 2012"
---

By **Jonathan Corbet**  
January 3, 2012

As a general rule, most developers feel that device drivers belong in the kernel. Kernel-space drivers are (hopefully) widely reviewed, implement standard device interfaces, perform better, and are more secure than the user-space variety. There are exceptions, though. Some high-performance applications want to talk to devices directly. Virtualized guests can also be thought of as a sort of user-space process; it is often desirable to allow guests to work with hardware directly rather than funneling their I/O through the host. So the kernel really should support this mode of access for the times when it is needed. 

The kernel's [UIO](<https://lwn.net/Articles/232575/>) interface has been available for the implementation of user-space drivers for some years. UIO has some shortcomings, though, including a lack of support for direct memory access (DMA) operations. DMA under user-space control is challenging to support for a number of reasons, not the least of which is security. A DMA-capable device is normally capable of writing _any_ page in memory; as a result, empowering a user-space process to set up DMA operations is equivalent to giving it full root access. Sometimes a user-space driver can be trusted with that access, but that is often not the case, especially when virtualization is involved. 

More recent CPUs have added support for safe (or safer) access to devices from virtualized guests. Devices can be restricted, via an I/O memory management unit (IOMMU) so that only specific regions of memory are accessible to them. Technologies like KVM support a "device assignment" mechanism that uses the hardware capabilities to hand a device to a guest, but device assignment is not without its shortcomings. Among other things, device assignment alone cannot guarantee the isolation of a specific device, and it involves a fair amount of complexity in the kernel. 

Alex Williamson's [VFIO patch set](<https://lwn.net/Articles/473975/>) is an attempt to come up with a better solution that allows the development of safe, high-performance user-space drivers. It provides interfaces allowing those drivers to work with DMA and interrupts while keeping overall control over how devices access the system's resources. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

One problem with KVM's device assignment is that it assumes that all devices are fully independent of each other. In particular, groups of devices may be connected through the same IOMMU; that means that any device can access any memory regions made available to any other devices in the same group. That, in turn, implies that the group of devices must be assigned as a unit; if any of those devices are assigned separately, the isolation of the group as a whole can be broken. 

So the first thing a VFIO driver writer will encounter is the group mechanism. The VFIO code creates the groups to match the hardware topology. It then ensures that every device in a group is controlled by a VFIO driver; if any device is unavailable, then the group as a whole cannot be used. Most devices on a typical system are unlikely to be bound to VFIO drivers at boot, so the system administrator must explicitly unbind them and tell VFIO to claim them. This is probably a good thing; exposing groups of devices to user space is best not done by default. 

For each group, a virtual device is created under `/dev/vfio`; prior to working with any individual device, a driver must open the group, claiming ownership of it. The access permissions on the group file control access to the underlying devices. Once the group has been opened, the driver should do an `ioctl(VFIO_GROUP_GET_INFO)` call to determine whether the group is "viable" (meaning all of the relevant devices are assigned to it) and available for use. If the group is not viable, the driver will not be able to proceed. 

To work with specific devices, the driver will "open" them with the `VFIO_GROUP_GET_DEVICE_FD` `ioctl()` call, which returns a file descriptor for access to the device. The `VFIO_DEVICE_GET_REGION_INFO` command can be used to learn about the device's memory-mapped I/O regions, which can then be accessed via an `mmap()` call. `VFIO_DEVICE_GET_IRQ_INFO` returns information about the device's interrupt assignment(s); the driver can use the `eventfd()` mechanism to receive notification of interrupts via a file descriptor. For most hardware, access to MMIO and interrupts is enough to communicate with the device. 

That still leaves the DMA problem, though. To that end, the `VFIO_GROUP_GET_IOMMU_FD` command returns a file descriptor representing the IOMMU. DMA mappings can be set up by filling in a `vfio_dma_map` structure: 

```
struct vfio_dma_map {
    	__u32	argsz;
    	__u32	flags;
    	__u64	vaddr;		/* Process virtual address */
    	__u64	iova;		/* IO virtual address */
    	__u64	size;		/* Size of mapping (bytes) */
        };
```

This structure is used to request a mapping of the user-space memory found at `vaddr` (of `size` bytes) into the device's I/O memory range starting at `iova`; the `VFIO_IOMMU_MAP_DMA` command actually gets the work done. For most user-space drivers, that should be about all that is needed, modulo a few details. 

Not all VFIO drivers will be in user space, though. Inside the kernel, VFIO looks like a special bus type to which devices can be bound. A VFIO driver needs to provide a set of operations to the core: 

```
struct vfio_device_ops {
    	bool	(*match)(struct device *dev, const char *buf);
    	int	(*claim)(struct device *dev);
    	int	(*open)(void *device_data);
    	void	(*release)(void *device_data);
    	ssize_t	(*read)(void *device_data, char __user *buf,
    			size_t count, loff_t *ppos);
    	ssize_t	(*write)(void *device_data, const char __user *buf,
    			 size_t count, loff_t *size);
    	long	(*ioctl)(void *device_data, unsigned int cmd,
    			 unsigned long arg);
    	int	(*mmap)(void *device_data, struct vm_area_struct *vma);
        };
```

Most of these operations are analogous to those found in `struct file_operations` or the bus-specific device structures. A device registered in this way can be opened and used like any other device with one difference: the interlock with group ownership is always enforced. If a device has been opened individually, the group is not "viable" and cannot be used by a user-space driver. If, instead, the group has been opened, the individual devices are busy and cannot be opened. 

VFIO is not the only patch set aimed at this problem; David Gibson's [device isolation infrastructure](<https://lwn.net/Articles/472269/>) is also intended to enable safe assignment of devices. The scope of this patch set is smaller, though, focusing mostly on the grouping aspect; there is no mechanism for controlling the IOMMU or working with individual devices. There is a certain amount of disagreement between the two on how grouping should be managed which suggests, in turn, that a certain amount of discussion will have to take place before either can be merged.
