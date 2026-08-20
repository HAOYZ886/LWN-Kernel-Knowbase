---
title: "Device-to-device memory-transfer offload with P2PDMA"
url: https://lwn.net/Articles/767281/
date: "October 2, 2018"
category: "Device drivers-Support APIs; PCI"
author: "October 2, 2018 This article was contributed by Marta Rybczyńska"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

October 2, 2018

This article was contributed by Marta Rybczyńska

One of the most common tasks carried out by device drivers is setting up DMA operations for data transfers between main memory and the device. Often, data read into memory from one device will be immediately written, unchanged, to another device. Common examples include carrying the image between the camera and screen on a mobile phone, or downloading files to be saved on a disk. Those transfers have an impact on the CPU even if it does not use the data directly, due to higher memory use and effects like cache trashing. There are cases where it is possible to avoid usage of the system memory completely, though. A [patch set](<https://lwn.net/Articles/764716/>) (posted by Logan Gunthorpe with contributions by Christoph Hellwig and Steve Wise) has been in the works for some time that addresses this case for PCI devices using peer-to-peer (P2P) transfers, with a focus on offering an offload option for the NVMe fabrics target subsystem.

#### PCI peer-to-peer memory concepts

PCI devices expose memory to the host system in form of memory regions defined by [base address registers](<https://wiki.osdev.org/PCI#Base_Address_Registers>) (BARs). Those are regions mapped into the host's physical memory space. All regions are mapped into the same address space, and PCI DMA operations can use those addresses directly. It is thus possible for a driver to configure a PCI DMA operation to perform transfers between the memory zones of two devices while bypassing system memory completely. The memory region might be on a third device, in which case two transfers are still required, but even in that case the advantage is lower load on the system CPU, decreased memory usage, and possibly lower PCI bandwidth usage. In the specific case of the [NVMe fabrics target [PDF]](<https://nvmexpress.org/wp-content/uploads/NVMe_over_Fabrics_1_0_Gold_20160605-1.pdf>), the data is transferred from a [remote direct memory access](<https://en.wikipedia.org/wiki/Remote_direct_memory_access>) (RDMA) network interface to a special memory region, then to the NVMe drive directly.

The difficulty is in obtaining the addresses and communicating them to the devices. This has been solved by introducing a new interface, called "p2pmem", that allows drivers to register suitable memory zones, discover zones that are available, allocate from them, and map them to the devices. Conceptually, drivers using P2P memory can play one or more of three roles: provider, client, and orchestrator: 

  * Providers publish P2P resources (memory regions) to other drivers. In the NVMe fabrics implementation, this is the done by the NVMe PCI driver that exports zones of the NVMe devices. 
  * Clients make use of the resources, setting up DMA transfers from and to them. In the NVMe fabrics implementation there are two clients: the NVMe PCI driver accepts buffers in P2P memory, and the RDMA driver uses it for DMA operations. 
  * Finally, orchestrators manage flows between providers and clients; in particular, they collect the list of available memory regions and choose the one to use. In this implementation there are also two orchestrators: NVMe PCI again, and the NVMe target that sets up the connection between the RDMA driver and the NVMe PCI device. 

Other scenarios are possible with the proposed interface; in particular, the memory region may be exposed by a third device. In this case two transfers will still be required, but without the use of the system memory.

#### Driver interfaces

For the provider role, registering device memory as being available for P2P transfers takes place using:

```
int pci_p2pdma_add_resource(struct pci_dev *pdev, int bar, size_t size,
    				u64 offset);
```

The driver specifies the parameters of the memory region (or parts of it). The zone will be represented by `ZONE_DEVICE` `page` structures associated with the device. When all resources are registered, the driver may publish them to make them available to orchestrators with:

```
void pci_p2pmem_publish(struct pci_dev *pdev, bool publish);
```

In the orchestrator role, the driver must create a list of all clients participating in a specific transaction so that a suitable range of P2P memory can be found. To that end, it should build that list with: 

```
int pci_p2pdma_add_client(struct list_head *head, struct device *dev);
```

The orchestrator can also remove the clients with `pci_p2pdma_remove_client()` and free the list completely with `pci_p2p_client_list_free()`:

```
void pci_p2pdma_remove_client(struct list_head *head, struct device *dev);
        void pci_p2pdma_client_list_free(struct list_head *head);
```

When the list is finished, the orchestrator can locate a suitable memory region available for all client devices with:

```
struct pci_dev *pci_p2pmem_find(struct list_head *clients);
```

The choice of provider is determined by its "distance", defined as the number of hops in the PCI tree between two devices. It is zero if the two devices are the same, four if they are behind the same switch (up to the downstream port of the switch, up to the common upstream, then down to the other downstream port and the final hop to the device). The closest (to all clients) suitable provider will be chosen; if there is more than one at the same distance, one will be chosen at random ([to avoid using the same one for all devices](<https://lwn.net/ml/linux-kernel/e9739430-b238-2078-a51f-d76bae5342d1%40deltatee.com/>)). Adding new clients to the list after locating the provider is possible if they are compatible; adding incompatible clients will fail.

There is a different path for the orchestrators that know which provider to use or that want to use different criteria for the choice. In such case, the driver should verify that the provider has available P2P memory with:

```
bool pci_has_p2pmem(struct pci_dev *pdev);
```

Then it can calculate the cumulative distance from its clients to the memory with:

```
int pci_p2pdma_distance(struct pci_dev *provider, struct list_head *clients,
    			    bool verbose);
```

When the orchestrator has found the desired provider, it can assign that provider to the client list using:

```
bool pci_p2pdma_assign_provider(struct pci_dev *provider,
        				    struct list_head *clients);
```

This call returns false if any of the clients are unsupported. After the provider has been selected, the driver can allocate and free memory for DMA transactions from that device using:

```
void *pci_alloc_p2pmem(struct pci_dev *pdev, size_t size);
        void pci_free_p2pmem(struct pci_dev *pdev, void *addr, size_t size);
```

Additional helpers exist for allocating scatter-gather lists with P2P memory:

```
pci_bus_addr_t pci_p2pmem_virt_to_bus(struct pci_dev *pdev, void *addr);
        struct scatterlist *pci_p2pmem_alloc_sgl(struct pci_dev *pdev, unsigned int *nents,
     					     u32 length);
        void pci_p2pmem_free_sgl(struct pci_dev *pdev, struct scatterlist *sgl);
```

While passing the P2P memory for DMA, the addresses must be PCI bus addresses. The users of the memory (clients) need to change their DMA mapping routine to:

```
int pci_p2pdma_map_sg(struct device *dev, struct scatterlist *sg, int nents,
    			  enum dma_data_direction dir);
```

A driver using P2P memory will use `pci_p2pmem_map_sg()` instead of `dma_map_sg()`. This routine is lighter, it just adjusts the bus offset, as the P2P uses bus addresses. To determine which mapping functions to use, drivers can benefit from this helper:

```
bool is_pci_p2pdma_page(const struct page *page);
```

#### Special properties

One of the most important tradeoffs the authors faced was finding out which hardware system configurations can be expected to work for P2P DMA operations. In PCI, each [root complex](<https://en.wikipedia.org/wiki/Root_complex>) defines its own hierarchy. Some complexes do not support peer-to-peer transfers between different hierarchies and there is no reliable way to find out if they do (see the PCI Express specification r4.0, section 1.3.1). The authors have decided to allow the P2P functionality only if all devices involved are behind the same PCI host bridge; otherwise the user would be required to understand their PCI topology and understand all devices in their system. This restriction may be lifted with time.

Even so, the configuration requires user intervention, as it is necessary to to pass the kernel parameter `disable_acs_redir` that was [introduced in 4.19](<https://lwn.net/ml/linux-kernel/20180730161840.13733-1-logang%40deltatee.com/>). This disables certain parts of the PCI access control services functionality that might redirect P2P requests (the low-level details have been deeply [discussed earlier](<https://lwn.net/ml/linux-kernel/20180423233046.21476-5-logang%40deltatee.com/>) in the development of this patch set).

P2P memories have special properties -- they are I/O memories without side effects (they are not device-control registers) and they are not cache coherent. The code handling those memories should be prepared and avoid passing this memory to code that is not. `iowrite*()` and `ioread*()` are not necessary, as there are no side effects, but if the driver needs a spinlock to protect its accesses, it should use `mmiowb()` before unlocking. There are currently no checks in the kernel to ensure the correct usage of this memory.

#### Other subsystem changes

Using P2P transfers with the NVMe subsystem required some changes in other subsystems, too. The block layer gained an additional flag, `QUEUE_FLAG_PCI_P2P`, to indicate that the specific queue can target P2P memory. A driver that submits a request using P2P memory should make sure that this flag is set on the target queue. There has been a discussion if there should be an additional check, but the developers [decided against it](<https://lwn.net/ml/linux-kernel/34d9b2f7-9e6d-4a0a-77e5-ec0e99610f5c@kernel.dk/>).

The NVMe driver was modified to use the new infrastructure; it also serves as an example of the implementation. The NVMe controller memory buffer (CMB) functionality, which is memory in the NVMe device that can be used to store commands or data, has been changed to use P2P memory. This means that, if P2P memory is not supported, the NVMe CMB functionality won't be available. The authors find that reasonable, since CMB is designed for P2P operations in the first place. Another change is that the request queues can benefit from P2P memory too.

RDMA, which is used for the NVMe fabrics, is now using flags to indicate if it should use P2P or regular allocations. The NVMe fabrics target itself allows the system administrator to choose to use P2P memory and specify the memory device using a configuration attribute that can be a boolean or PCI device name. In the first case it will use any suitable P2P memory, in the second -- only from the specific P2P memory device.

#### Current state

The patch set has been under review for months now (see [this presentation [PDF]](<https://www.snia.org/sites/default/files/SDC/2018/presentations/Storage_Architecture/Bates_Stephen_Accelerating_Storage_with_NVM_Express_SSDs_and_P2PDMA.pdf>)), and the authors provide a long list of hardware it has been tested with. The pace of this patch set (up to [version 8](<https://lwn.net/ml/linux-kernel/20180927165420.5290-1-logang@deltatee.com>) as of this writing) is fast; it seems that it might be merged in the near future. 

The patch set allows use cases that were not possible with the mainline kernel before and opens a window for other use cases (P2P can be used with graphics cards, for example). At this stage, the support is basic and there are numerous modifications and extensions to be added in the future; one direction will be to extend the range of supported configurations. Others would be to hide the API behind the generic DMA operations and use the optimization with other types of devices.
