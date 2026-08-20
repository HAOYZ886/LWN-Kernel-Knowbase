---
title: Managing multifunction devices with the auxiliary bus
url: https://lwn.net/Articles/840416/
date: "December 17, 2020"
category: "Auxiliary bus; Device drivers-Support APIs"
author: "December 17, 2020 This article was contributed by Marta Rybczyńska"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

December 17, 2020

This article was contributed by Marta Rybczyńska

Device drivers usually live within a single kernel subsystem. Sometimes, however, developers need to handle functionalities outside of this model. Consider, for example, a network interface card (NIC) exposing both Ethernet and RDMA functionalities. There is one hardware block, but two drivers for the two functions. Those drivers need to work within their respective subsystems, but they must also share access to the same hardware. There is no standard way in current kernels to connect those drivers together, so developers invent ad-hoc methods to handle the interaction between them. Recently, Dave Ertman [posted](<https://lwn.net/ml/netdev/20201113161859.1775473-1-david.m.ertman@intel.com/>) a patch set introducing a new type of a bus, called the "auxiliary bus", to address this problem.

#### Complex devices

Linux already includes a number of drivers for multi-function devices. One of the ways to support them is the [Multi-Function Devices](<https://www.kernel.org/doc/Documentation/devicetree/bindings/mfd/mfd.txt>) (MFD) subsystem. It handles independent devices "glued" together into one hardware block which may contain some shared resources. MFD allows access to device registers either directly, or using a common bus. In this second case, it conveniently multiplexes accesses on [Inter-Integrated Circuit (I2C)](<https://en.wikipedia.org/wiki/I%C2%B2C>) or [Serial Peripheral Interface (SPI)](<https://en.wikipedia.org/wiki/Serial_Peripheral_Interface>) buses. As the MFD sub-devices are separate, MFD drivers do not share a common state.

The devices Ertman addresses do not fit well into the MFD model. Devices using the auxiliary bus provide subsets of the capabilities of a single hardware device. They do not expose separate register sets for each function; thus they cannot be described by devicetrees or discovered by ACPI. Their drivers need to share access to the hardware. Events concerning all sub-functionalities (like power management) need to be properly handled by all drivers. These devices will often be specialized processors running firmware and communicating with the host system (and the Linux drivers) by messaging. The available functions may not be known in advance, and thus must be discovered at run time. 

The [documentation patch](<https://lwn.net/ml/netdev/20201113161859.1775473-2-david.m.ertman@intel.com/>) in the auxiliary bus series cites a number of examples. The [Sound Open Firmware (SOF) driver](<https://elixir.bootlin.com/linux/v5.9.11/source/sound/soc/sof>) interacts with a single device exposing interfaces like HDMI output, microphones, speakers, testing, and debug hooks. NICs implementing both Ethernet and RDMA may need a driver supporting a common part of the functionalities, and then the specific Ethernet and RDMA drivers can implement specific parts on top of that.

Current kernels do not have a generic way to describe dependencies between drivers for this kind of device. A solution to the problem could be to have a way to attach secondary drivers to the primary one; this is exactly what the auxiliary bus implements.

#### Auxiliary devices and drivers

The patch set introduces two main concepts: The "auxiliary device" and "auxiliary driver". These implement the relationship between the main and the secondary drivers. The main driver maintains the device state, allocating and managing all shared data. It also unregisters all secondary drivers when shutting down. Secondary drivers, instead, handle the interactions with the specific subsystem they are implementing a device for.

Each main driver may expose a number of functionalities (devices) for secondary drivers. Only one secondary driver can attach to each of those functionalities.

The main driver creates an auxiliary device, represented by `struct auxiliary_device`: 

```
struct auxiliary_device {
            struct device dev;
            const char *name;
            u32 id;
        };
```

The combination of `name` and `id` must be unique; the complete device name is a combination of the module name and those two fields, connected by dots (`.`). That yields a result like `modname.device_name.id`. 

The developer embeds this structure in the `device` structure of the main driver, with all shared data necessary for the communication between the main driver and secondary drivers. They may also add supplementary callbacks.

The sequence to initialize the main driver contains two steps. The first one is to call `auxiliary_device_init()`:

```
int auxiliary_device_init(struct auxiliary_device *auxdev);
```

It verifies the arguments and returns an error code if need be; in such case the initialization of the device should be aborted. If the first call succeeds, the second step is to call the macro `auxiliary_device_add()` with the initialized device; this will set up the device name and register the device itself.

The unregistration procedure also has two steps, consisting of calls to `auxiliary_device_uninit()` (necessary from the point when `auxiliary_device_init()` has succeeded) and `auxiliary_device_delete()`. Those functions have the following prototypes:

```
void auxiliary_device_uninit(struct auxiliary_device *auxdev);
        void auxiliary_device_delete(struct auxiliary_device *auxdev);
```

This two-step approach was implemented [in response to comments](<https://lwn.net/ml/netdev/DM6PR11MB284123995577294BE3E0C36EDD0B0@DM6PR11MB2841.namprd11.prod.outlook.com/>) on earlier versions of the patch set. It allows the driver to allocate its own data between `auxiliary_device_init()` and `auxiliary_device_add()` with a possibility to free it correctly in the case of a failure.

The secondary devices, which will connect to the main driver, are represented by `struct auxiliary_driver`: 

```
struct auxiliary_driver {
            int (*probe)(struct auxiliary_device *auxdev,
    		     const struct auxiliary_device_id *id);
            int (*remove)(struct auxiliary_device *auxdev);
            void (*shutdown)(struct auxiliary_device *auxdev);
            int (*suspend)(struct auxiliary_device *auxdev, pm_message_t state);
            int (*resume)(struct auxiliary_device *auxdev);
            const char *name;
            struct device_driver driver;
            const struct auxiliary_device_id *id_table;
        };
```

This structure includes a number of callbacks to manage the device's life cycle, and the `id_table` containing names of the devices the driver can bind with. All callbacks receive pointers to the parent's `auxiliary_device`, allowing access to the shared data.

The secondary devices are set up with `auxiliary_driver_register()`:

```
int auxiliary_driver_register(struct auxiliary_driver *auxdrv);
```

This function requires the `probe()` callback and the `id_table` to be filled in. When successful, it causes a `probe()` callback call for any matching devices. The secondary devices can access the shared data using `container_of()` and the `auxiliary_device` structure.

When unregistering a driver, the developer should call `auxiliary_driver_unregister()`: 

```
void auxiliary_driver_unregister(struct auxiliary_driver *auxdrv);
```

#### First users

Together with the auxiliary bus implementation, Ertman [posted changes](<https://lwn.net/ml/netdev/20201113161859.1775473-3-david.m.ertman@intel.com/>) to the SOF driver. The modified driver uses this infrastructure to [implement a test driver](<https://lwn.net/ml/netdev/20201113161859.1775473-4-david.m.ertman@intel.com/>), and [a probes driver](<https://lwn.net/ml/netdev/20201113161859.1775473-10-david.m.ertman@intel.com/>), allowing the creation of a new virtual audio device that can tap into the pipeline and allow listening in at any point.

Another user can be found in the networking subsystem; Leon Romanovsky [posted](<https://lwn.net/ml/netdev/20201026111849.1035786-1-leon%40kernel.org/>) a conversion of the mlx5 driver to use the auxiliary bus. The updated driver creates network, [VDPA](<https://lwn.net/Articles/828042/>), and RDMA drivers for one physical device. Those changes [allow the removal](<https://lwn.net/ml/netdev/20201026111849.1035786-10-leon@kernel.org/>) of a bunch of custom driver code. Parav Pandit [followed up](<https://lwn.net/ml/netdev/20201112192424.2742-1-parav%40nvidia.com/>) by using this functionality to implement device sub-functions.

The patch set has come to its fourth iteration in its current form, and witnessed a number of earlier ones under the names of [ancillary](<https://lwn.net/ml/netdev/20201001050851.890722-1-david.m.ertman@intel.com/>) and [virtual](<https://lwn.net/ml/netdev/20191111192219.30259-1-jeffrey.t.kirsher%40intel.com/>) bus. The development of the auxiliary bus patch set took time, and it created dependencies in other work. This caused a fair amount of pressure to get it upstream, and that led to some [pushing](<https://lwn.net/ml/netdev/20201117053000.GM47002@unreal/>) on the list. In an attempt to push things forward, Dan Williams [reposted the patch set](<https://lwn.net/ml/netdev/160695681289.505290.8978295443574440604.stgit@dwillia2-desk3.amr.corp.intel.com/>), stating that ""it looks good to me and several other stakeholders"". After [a review](<https://lwn.net/ml/netdev/X8ogtmrm7tOzZo+N@kroah.com/>) from Greg Kroah-Hartman, the auxiliary bus code was merged into the mainline for the 5.11 kernel release.
