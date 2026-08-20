---
title: Greybus
url: https://lwn.net/Articles/715955/
date: "March 1, 2017"
category: Greybus
author: "March 1, 2017 This article was contributed by Viresh Kumar"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

March 1, 2017

This article was contributed by [Viresh Kumar](<mailto:viresh.kumar@linaro.org>)

The Linux kernel gained a new subsystem during the [4.9](<https://lwn.net/Articles/708766/>) development cycle. That subsystem is Greybus and this article will briefly take you through its internals.

Greybus was initially designed for Google's [Project Ara](<https://atap.google.com/ara/>) smartphone (which is [discontinued](<http://www.theverge.com/2016/9/2/12775922/google-project-ara-modular-phone-suspended-confirm>) now), but the first (and only) product released with it is Motorola's [Moto Mods](<https://www.motorola.com/us/moto-mods>). There are discussions going on to evaluate the feasibility of using the protocols provided by Greybus in applications like the [Internet of things](<https://en.wikipedia.org/wiki/Internet_of_things>), and for other parts of the kernel that need to communicate in a platform-independent way.

Initially, Greg Kroah-Hartman tried to merge Greybus core in the kernel's `drivers` directory but, after some objections (people wanted to do more detailed reviews before merging it) everyone agreed to merge it into the staging tree instead. Almost 2400 patches, developed over 2.5 years, were merged; these contributions came from over 50 developers representing at least five organizations (Google, [Linaro](<https://www.linaro.org/>), [BayLibre](<http://baylibre.com/>), [LeafLabs](<http://www.leaflabs.com/>), and [MMSolutions](<https://www.mm-sol.com/>)). There were a lot more developers and companies involved in the development of the other parts of the Ara software and hardware. Greybus developers also showed up in the [list](<https://lwn.net/Articles/708266/>) of most active developers (the top four by changesets) for the 4.9 release.

Kroah-Hartman made sure that Greybus was merged with all its history preserved, [saying](<https://lwn.net/Articles/700618/>):

Because this was 2 1/2 years of work, with many many developers contributing, I didn't want to flatten all of their effort into a few small patches, as that wouldn't be very fair. So I've built a git tree with all of the changes going back to the first commit, and merged it into the kernel tree, just like btrfs was merged into the kernel. 

Jonathan Corbet wrote an earlier [article](<https://lwn.net/Articles/648400/>) on Greybus; readers may want to look at that to catch up on some history.

#### UniPro and the internals of the Greybus subsystem

The Project Ara smartphone was designed to be customizable. The user could select a subset from a wide range of modules, providing interesting capabilities (like cameras, speakers, batteries, displays, sensors, etc.), and attach them to the frame of the phone. The modules could communicate with the main processors or other modules directly over the [UniPro](<http://mipi.org/specifications/unipro-specifications>) bus. The specification of this bus is managed by the [MIPI](<http://mipi.org/>) alliance. UniPro follows the architecture of the classical OSI network model, except that it has no application layer defined. And that's where Greybus fits in. 

UniPro communication happens over bidirectional connections between entities, like the modules on the Ara smartphone; it doesn't need to go through the processors. Each UniPro device has virtual ports within it, which can be seen as sub-addresses within the device. They are a lot like sockets and are called "connection ports" (or CPorts). There is a switch on the bus that sets up the actual routes. Messages can pass at a rate of around 10Gb/s; the bus also has message prioritization, error handling, and notification of delivery problems, though UniPro doesn't support streams or multicast delivery.

As the Greybus specification was initially written for the Project Ara smartphone, it is greatly inspired by Ara's design, where modules can be inserted into or removed from the phone's frame on the fly. A lot of effort has been put in to make the specification as generic as possible, in order to make it fit for other use cases as well. You will also notice a lot of similarities with the [USB](<https://en.wikipedia.org/wiki/USB>) framework in the Linux kernel, as it was taken as a reference during the development of Greybus.

The Greybus specification provides device discovery and description at runtime, network routing and housekeeping, and class and bridged PHY protocols, which devices use to talk to each other and to the processors. The following figure gives a glimpse of how various parts of the kernel interact with the Greybus subsystem.

> ![\[Greybus diagram\]](https://static.lwn.net/images/2017/greybus.png)

The Greybus core implements the supervisory controller (SVC) protocol (described later), which is used by the application processor (AP — the CPUs running Linux) to communicate to the SVC. The SVC represents an entity within the Greybus network that configures and controls the Greybus (UniPro) network, mostly based on the instructions from the AP. All module insertion and removal events are first reported to the SVC, which in turn informs the AP about them using the SVC protocol. The AP is responsible for administrating the Greybus network via the SVC.

During initial development of the Ara smartphone, there were no SoCs available with built-in UniPro support. Separate hardware entities were designed to connect the AP to the UniPro network. These entities receive a message from AP which they translate and send to the UniPro network. The same was also required in the other direction: receive messages from UniPro and translate them to the AP. These entities are called as AP Bridge (APB) host controllers. They can receive messages over USB and send them over UniPro and vice versa. The AP isn't part of the Greybus network really and so isn't represented in the above picture. The Greybus subsystem also supports processors with built-in UniPro support; they are represented by native UniPro host controllers. The AP can talk directly to them without the USB subsystem.

During module initialization (after the module is detected on Greybus), the Greybus core parses the module's manifest, which describes the capabilities of the module, and creates devices within the kernel to represent it.

Power management for the whole UniPro network (i.e. AP, SVC, and modules) is managed by the Greybus core. During system suspend, the Greybus core puts the SVC and the modules into low-power states and, on system resume, it brings up the Greybus network. The Greybus core also performs runtime power management for all individual entities. For example, if a module isn't being used, the Greybus core will power it off and will bring it back only when it is required.

The Greybus core also binds itself to the Linux kernel driver core and provides a sysfs interface at `/sys/bus/greybus`. The following diagram depicts the sysfs hierarchy for a single AP Bridge (APB) connected to the AP. A single module is accessible via the APB and the module presents a single interface that contains two bundles (devices) within it. The figure also represents the control CPort per interface and the SVC per APB, along with a list of attributes for each entity. All of these entities will be described later in detail.

> 

```
>     greybus/
>     └── greybus1 (AP Bridge)
>         ├── 1-2 (Module)
>         │   ├── 1-2.2 (Interface)
>         │   │   ├── 1-2.2.1 (Bundle)
>         │   │   │   ├── bundle_class
>         │   │   │   ├── bundle_id
>         │   │   │   └── state
>         │   │   ├── 1-2.2.2 (Bundle)
>         │   │   │   ├── bundle_class
>         │   │   │   ├── bundle_id
>         │   │   │   └── state
>         │   │   ├── 1-2.2.ctrl (Control CPort)
>         │   │   │   ├── product_string
>         │   │   │   └── vendor_string
>         │   │   ├── ddbl1_manufacturer_id
>         │   │   ├── ddbl1_product_id
>         │   │   ├── interface_id
>         │   │   ├── product_id
>         │   │   ├── serial_number
>         │   │   └── vendor_id
>         │   ├── eject
>         │   ├── module_id
>         │   └── num_interfaces
>         ├── 1-svc (SVC)
>         │   ├── ap_intf_id
>         │   ├── endo_id
>         │   └── intf_eject
>         └── bus_id
>
```

The functionality provided by the modules is exposed using device-class and bridged PHY drivers. The device-class drivers implement protocols whose purpose is to provide a device abstraction for the functionality commonly found on the mobile handsets. For example, cameras, batteries, sensors, etc. The bridged PHY drivers, instead, implement protocols whose purpose is to support communication with the modules on the Greybus network that do not comply with a device-class protocol; these include integrated circuits using alternative physical interfaces to UniPro. For example, devices connected via GPIO, I2C, SPI, USB, etc. The modules that only implement device-class protocols are said to be device-class conformant. Modules that implement any of the bridged PHY protocols are said to be non-device-class conformant. The device classes and bridged PHY protocols will be listed later.

#### Module hierarchy

A module is the physical hardware entity that can be connected or disconnected statically (before powering the system on) or dynamically (while the system is running) from the Greybus network. Once the modules are connected to the Greybus network, the AP and the SVC enumerate the modules and fetch per-interface manifests to learn about their capabilities. The following figure gives a glimpse of how the module hierarchy looks in the Greybus subsystem:

> ![Module hierarchy](https://static.lwn.net/images/2017/greybus-modules.png)

Modules are represented within the Linux kernel by a `struct gb_module`:

> 

```
>     struct gb_module {
>         struct device dev;
>         u8 module_id;
>         size_t num_interfaces;
>         struct gb_interface *interfaces[0];
>         ...
>     }
>
```

Here, `dev` is the module's device structure, `module_id` is a unique eight-bit number assigned to the module by the SVC, `interfaces` points to the interfaces present within the module, and `num_interfaces` is their count.

The Greybus modules have electrical connectors on them, connecting them to the phone's frame. These electrical connectors are called "interface blocks" and are represented in software by the term "interface". A module can have one or more interfaces. The interface with the smallest interface ID is configured as the primary interface and all other are called secondary interfaces. The `module_id` is set to the ID of the primary interface. 

The primary interface is special as the AP receives module insertion event with the ID of the primary interface and the module can only be ejected from the frame using the primary interface. The interfaces can present any number of functionalities, which can be supported with the bandwidth available to the respective interface block. The interfaces are represented within the Linux kernel by the `struct gb_interface`:

> 

```
>     struct gb_interface {
>         struct device dev;
>         struct gb_control *control;
>         struct list_head bundles;
>         struct list_head manifest_descs;
>         u8 interface_id;
>         struct gb_module *module;
>         ...
>     };
>
```

Here, `dev` is the interface's device structure, `control` represents the control connection (described below), `bundles` is the list containing bundles within the interface, `manifest_descs` is the lists of descriptors created from the interface manifest, `interface_id` is the unique ID of the interface, and `module` is the pointer to the parent module structure. The module ID and interface ID both start from zero and are unique within the Greybus network.

The Greybus Interfaces can contain one or more bundles, each of which represents a logical Greybus device in the kernel. For example, an interface with vibrator and battery functionalities will have two bundles, one for the vibrator and one for the battery. Each bundle will get a `struct device` for itself and a greybus driver will bind to that device. The bundle ID is unique within an interface. The bundles are represented within the Linux kernel by the `struct gb_bundle`:

> 

```
>     struct gb_bundle {
>         struct device           dev;
>         struct gb_interface     *intf;
>         u8                      id;
>         u8                      class;
>         size_t                  num_cports;
>         struct list_head        connections;
>         ...
>     }
>
```

Here, `dev` is the bundle's device structure, `intf` is the pointer to the parent interface, `id` is the unique ID of the bundle within the interface, `class` is the class type of the bundle (like, camera or audio), `connections` is the connections within the bundle, and `num_cports` is the count of the connections.

The Greybus driver is represented by the following structure and it accepts the bundle structure as argument to all its callbacks:

> 

```
>     struct greybus_driver {
>         const char *name;
>         int (*probe)(struct gb_bundle *bundle,
>                      const struct greybus_bundle_id *id);
>         void (*disconnect)(struct gb_bundle *bundle);
>         const struct greybus_bundle_id *id_table;
>         struct device_driver driver;
>     };
>
```

Here, `name` is the name of the Greybus driver, `probe` and `disconnect` are the callbacks, `id_table` is the device bundle ID table, and `driver` is the generic device driver structure.

The Greybus or UniPro "connection" is a bidirectional communication path between two CPorts. There can be one or more CPorts within a bundle. The communication over the connections is governed by a predefined set of operations and the semantics of those operations is defined by the Greybus protocols (covered later). Each CPort is managed by exactly one protocol. The CPort numbers are unique within an interface. The first CPort within the interface is always the control CPort (CPort0, which is not part of any bundle) while the rest of the CPorts are numbered starting with one. CPort0 is special and is used for the management of its interface. It is governed by a special protocol, the control protocol (described later). The connections are represented within Linux kernel by the `struct gb_connection`:

> 

```
>     struct gb_connection {
>         struct gb_host_device           *hd;
>         struct gb_interface             *intf;
>         struct gb_bundle                *bundle;
>         u16                             hd_cport_id;
>         u16                             intf_cport_id;
>         struct list_head                operations;
>         ...
>     };
>
```

Here, `hd` represents the AP bridge through which the AP communicates with the module, `intf` represents the parent interface, `bundle` represents the parent bundle, `hd_cport_id` represents the CPort ID of the AP bridge, `intf_cport_id` represents the CPort ID of the interface, and `operations` is the list of operations that are getting exchanged over the connection. The connection is established between `hd_cport_id` and `intf_cport_id`.

Greybus bundles can also represent complex functionalities, such as audio or camera. Normally, such complex devices consist of multiple components working together, like sensors, DMA controllers, bridges, audio interfaces, codecs, etc., and a single bundle device may look insufficient to represent them all. But that's how Greybus represents such devices. The module side contains the firmware that makes all these components work together; it takes inputs from the AP over the connections present within the bundle. For example, a bundle representing the camera will have two connections: data and management. All management instructions are sent to the module or configurations are received from the module using the management connection. And the data from the camera on the module is received over the data connection. The internals of how various components work together to represent the camera are hidden from Greybus and hence the AP.

When a module and its interfaces are connected to the Greybus network (by attaching the module to the frame of the phone), the AP starts enumerating its interfaces over CPort0. The AP fetches a block of data from the interfaces, called the interface manifest. The manifest is a data structure containing the manifest header along with a set of descriptors. The manifest allows the AP to learn about the capabilities of the interface.

Following is a simple example of a raw manifest file that represents an interface that supports a single audio bundle. The manifest file is converted into a binary blob using the [Manifesto library](<https://github.com/projectara/manifesto>). In the following example, the bundle has two connections: "Management" and "Data". Note that it is optional to add the control CPort0 in the manifest file.

> 

```
>     ; Simple Audio Interface Manifest
>     ;
>     ; Provided under the three clause BSD license found in the LICENSE file.
>     
>     [manifest-header]
>     version-major = 0
>     version-minor = 1
>     
>     [interface-descriptor]
>     vendor-string-id = 1
>     product-string-id = 2
>     
>     ; Interface vendor string
>     [string-descriptor 1]
>     string = Project Ara
>     
>     ; Interface product string
>     [string-descriptor 2]
>     string = Simple Audio Interface
>     
>     ; Bundle 1: Audio class
>     [bundle-descriptor 1]
>     class = 0x12
>     
>     ; Audio Management protocol on CPort 1
>     [cport-descriptor 1]
>     bundle = 1
>     protocol = 0x12
>     
>     ; Audio Data protocol on CPort 2
>     [cport-descriptor 2]
>     bundle = 1
>     protocol = 0x13
>
```

#### Greybus messages

Greybus communication is built on UniPro messages, which are used to exchange information between the AP, SVC, and the modules. Normally all communication is bidirectional, in that for every request message from a sender, the receiver responds with a response message. Which entity (AP, SVC, or the module) can initiate a request message depends on the individual operation as defined by the respective protocol. For example, only the AP can initiate operations on the control protocol. Some of the operations are unidirectional operations as well, where the receiver doesn't need to generate a response message.

Each message sent over UniPro begins with a short header, followed by operation-specific payload data. The message header is represented by following structure:

> 

```
>     struct gb_operation_msg_hdr {
>         __le16  size;           /* Size in bytes of header + payload */
>         __le16  operation_id;   /* Operation unique id */
>         __u8    type;           /* E.g GB_I2C_TYPE_TRANSFER */
>         __u8    result;         /* Result of request (in responses only) */
>         __u8    pad[2];         /* must be zero (ignore when read) */
>     }
>
```

Here, `size` is the size of the header (8 bytes) plus size of the payload data. The size of payload data is defined by each operation of every protocol. The `operation_id` is a unique, 16-bit number which is used to match request and response messages. The `operation_id` allows many operations to be "in flight" on a connection at once. The special ID zero is reserved for unidirectional operations. The operation `type` is an eight-bit number that defines the type of the operation. The meaning of the type value depends on the protocol in use. Only 127 operations are available for a given protocol (0x01..0x7f); operation 0x00 is reserved. The most significant bit (0x80) of the operation type is used as a flag that distinguishes a request message from its response. For requests, this bit is zero, for responses, it is one. The `result` field is ignored for request messages; it contains the result of the requested operation in the response message.

Greybus messages (both request and response) are managed using the following structure within the Linux kernel:

> 

```
>     struct gb_message {
>         struct gb_operation             *operation;
>         struct gb_operation_msg_hdr     *header;
>         void                            *payload;
>         size_t                          payload_size;
>         ...
>     };
>
```

Here, `operation` is the operation to which the message belongs, `header` is the header to be sent over UniPro, `payload` is the payload to be sent following the header, and `payload_size` is the size of the payload.

An entire Greybus operation (a request and its response) is managed using the following structure within the Linux kernel:

> 

```
>     struct gb_operation {
>         struct gb_connection    *connection;
>         struct gb_message       *request;
>         struct gb_message       *response;
>         u8                      type;
>         u16                     id;
>         ...
>     };
>
```

Here, `connection` represents the communication path over which UniPro messages are sent, `request` and `response` represent the Greybus messages, and `type` and `id` are as described earlier in the message header.

There are multiple helpers which can be used to send/receive Greybus messages over a connection, but most of the users end up using the following:

> 

```
>     int gb_operation_sync_timeout(struct gb_connection *connection, int type,
>                                   void *request, int request_size, void *response,
>                                   int response_size, unsigned int timeout);
>
```

Here, `connection` represents the communication path over which UniPro messages are sent, `type` is the operation type, `request` is the request payload, `request_size` is the size of the request payload, `response` is the space for response payload, `response_size` is the size of the expected response payload, and `timeout` is the timeout period for the operation in milliseconds. Usually, a timeout of 1ms is chosen.

`gb_operation_sync_timeout()` creates the operation and its messages, copies the request payload into the request message, and sends the request message header and its payload over the Greybus connection. It then waits for `timeout` milliseconds for a response from the other side and errors out if no response is received within that time. Otherwise, once the response is received, it first checks the response header to check the result of the operation. If result field indicates an error, then `gb_operation_sync_timeout()` errors out. Otherwise, it copies the response payload into the memory pointed by the `response` field and destroys the operation and message structures. It returns zero on success or a negative number to represent errors.

#### Greybus protocols

The Greybus protocols define the layout and semantics of messages that are exchanged over a Greybus connection. Each Greybus protocol defines a set of operations, with formats of their request and response messages. It also defines which side of the connection can initiate each request. The Greybus protocols are broadly divided into three categories: special protocols, device-class protocols, and bridged PHY protocols.

The special protocols are the core Greybus protocols with administrative powers over the Greybus network. There are two special protocols: SVC and Control.

The SVC protocol serves the purpose of communication between the AP and the SVC. The AP controls the network via the SVC using this protocol. The CPort0 of the APB is used for the SVC connection (don't confuse that with the CPort0 of each module interface which is used for the control protocol). The main purpose of this protocol is to help the AP create routes between various CPorts, sense module insertion or removal, etc. The modules on the Greybus network should not implement it. Operations defined under this protocol include module insertion and removal events, the creation and destruction of routes and connections, and more. 

The control protocol serves the purpose of communication between the AP and the module's interfaces. The AP controls individual interfaces using this protocol. The main purpose of this protocol is to help the AP enumerate a new interface and learn about its capabilities. Only the AP can initiate operations (send requests) under this protocol and the module must respond to those requests. Some of the operations allowed under this protocol can get an interface's manifest, or activate, deactivate, suspend, or resume a bundle. 

As mentioned earlier, the device-class protocols provide a device abstraction for the functionality commonly found on mobile handsets. A simple example of that can be the audio management protocol or the camera management protocol. The bridged PHY protocols provide communication with the modules on the Greybus network that do not comply with an existing device class protocol, and which include integrated circuits using alternative physical interfaces to UniPro. 

#### What's next

It would be interesting to do a couple of things going forward. To begin with, the Greybus core should be moved from the staging tree to the kernel's `drivers` directory. The drivers themselves should be moved to their respective frameworks: `.../greybus/gpio.c` would become `.../gpio/gpio-greybus.c`, for example. This would take reasonable amount of time and effort. 

Later on, it would be nice to get Motorola's Moto Mods support merged into the kernel and include its improvements to the Greybus subsystem. That will mostly depend on the Motorola community, though. As Project Ara is discontinued now, it would be quite interesting to find new targets (like the Internet of things) for the Greybus subsystem and adapt Greybus to support them. Some discussions are going on about that.
