---
title: "Driver porting: Network drivers"
url: https://lwn.net/Articles/30107/
date: "April 28, 2003"
category: NAPI
author: 
---

This article is part of the LWN [Porting Drivers to 2.6 series](<https://lwn.net/Articles/driver-porting/>).   
---  
Much of the core network driver API has not been changed between the 2.4 and 2.6 kernels. With only a relatively small amount of work, most drivers should function just fine under 2.6. If, however, you want to get the very best performance out of high-bandwidth network cards, you may have to make more extensive changes to your driver to work with the new APIs which have been made available. 

### Network device allocation

In 2.6, network devices are part of the wider kernel device model. There are advantages to this change, including the fact that network device information is available under `/sys/class/net/`. But hooking into the driver model poses a new set of potential race conditions which were not there before. What happens if your driver module is removed while a process has an associated sysfs file open? Network drivers are more susceptible than most to this problem because the networking subsystem does not restrict the unloading of drivers via the module use count. 

The only way to properly deal with this problem is to allocate network devices in a dynamic manner, and to let the device model code figure out when to free them. To that end, all `net_device` structures must be allocated with the new `alloc_netdev()` function: 

```
struct net_device *alloc_netdev(int sizeof_priv, const char *name,
    			       	    void (*setup)(struct net_device *));
```

Here, `sizeof_priv` is the size of the structure that you would otherwise allocate and assign to the `net_device` `priv` field; `alloc_netdev()` will allocate that memory for you as well. `name` is the name of the device (a format string is acceptable, so something like `"eth%d"` works), and `setup` is a function to be called to complete the initialization of the `net_device` structure. The `setup` function can be the same function that, in older drivers, you may have assigned to the `init` field in the `net_device` structure. 

For Ethernet devices, there is a simpler form: 

```
struct net_device *alloc_etherdev(int sizeof_priv);
```

Calling this function is equivalent to: 

```
my_dev = alloc_netdev(sizeof(my_priv), "eth%d", setup_ether);
```

Either way, when you are done with the device (i.e. after you have called `unregister_netdev()`), you must free it with: 

```
void free_netdev(struct net_device *dev);
```

Note that it would be an error to free the `priv` field separately - let `free_netdev()` take care of it. 

### NAPI

The most significant change, perhaps, is the addition of NAPI ("New API"), which is designed to improve the performance of high-speed networking. NAPI works through: 

  * Interrupt mitigation. High-speed networking can create thousands of interrupts per second, all of which tell the system something it already knew: it has lots of packets to process. NAPI allows drivers to run with (some) interrupts disabled during times of high traffic, with a corresponding decrease in system load. 

  * Packet throttling. When the system is overwhelmed and must drop packets, it's better if those packets are disposed of before much effort goes into processing them. NAPI-compliant drivers can often cause packets to be dropped in the network adapter itself, before the kernel sees them at all. 

  * More careful packet treatment, with special care taken to avoid reordering packets. Out-of-order packets can be a significant performance bottleneck. 

NAPI was also backported to the 2.4.20 kernel. 

The following is a whirlwind tour of what must be done to create a NAPI-compliant network driver. More details can be found in `networking/NAPI_HOWTO.txt` in the kernel documentation directory, and, of course, in the source of drivers which have been converted. Note that use of NAPI is entirely optional, drivers will work just fine (though perhaps a little more slowly) without it. 

The first step is to make some changes to your driver's interrupt handler. If your driver has been interrupted because a new packet is available, that packet should not be processed at the time. Instead, your driver should disable any further "packet available" interrupts and tell the networking subsystem to poll your driver shortly to pick up all available packets. Disabling interrupts, of course, is a hardware-specific matter between the driver and the adaptor. Arranging for polling is done with a call to: 

```
void netif_rx_schedule(struct net_device *dev);
```

An alternative form you'll see in some drivers is: 

```
if (netif_rx_schedule_prep(dev))
            __netif_rx_schedule(dev);
```

The end result is the same either way. (If `netif_rx_schedule_prep()` returns zero, it means that there was already a poll scheduled, and you should not have received another interrupt). 

The next step is to create a `poll()` method for your driver; it's job is to obtain packets from the network interface and feed them into the kernel. The `poll()` prototype is: 

```
int (*poll)(struct net_device *dev, int *budget);
```

The `poll()` function should process all available incoming packets, much as your interrupt handler might have done in the pre-NAPI days. There are some exceptions, however: 

  * Packets should not be passed to `netif_rx()`; instead, use: 

```
int netif_receive_skb(struct sk_buff *skb);
```

The return value will be `NET_RX_DROP` if the networking subsystem had to drop the packet. Network drivers could use that information to stop feeding packets for the moment, but no driver in the kernel tree does so currently. 

  * A new `struct net_device` field called `quota` contains the maximum number of packets that the networking subsystem is prepared to receive from your driver at this time. Once you have exhausted that quota, no further packets should be fed to the kernel in this `poll()` call. 

  * The `budget` parameter also places a limit on the number of packets which your driver may process. Whichever of `budget` and `quota` is lower is the real limit. 

  * Your driver should decrement `dev->quota` by the number of packets it processed. The value pointed to by the `budget` parameter should also be decremented by the same amount. 

  * If packets remain to be processed (i.e. the driver used its entire quota), `poll()` should return a value of one. 

  * If, instead, all packets have been processed, your driver should reenable interrupts, turn off polling, and return zero. Polling is stopped with: 

```
void netif_rx_complete(struct net_device *dev);
```

The networking subsystem promises that `poll()` will not be invoked simultaneously (for the same device) on multiple processors. 

The final step is to tell the networking subsystem about your `poll()` method. This, of course, is done in your initialization code when all the other `struct net_device` fields are set: 

```
dev->poll = my_poll;
        dev->weight = 16;
```

The `weight` field is a measure of the importance of this interface; the number stored here will turn out to be the same number your driver finds in the `quota` field when `poll()` is called. If you forget to initialize `weight` and leave it at zero, `poll()` will never be called (voice of experience here). Gigabit adaptor drivers tend to set `weight` to 64; smaller values can be used for slower media. 

### Receiving packets in non-interrupt mode

Network drivers tend to send packets into the kernel while running in interrupt mode. There are occasions where, instead, packets will be received by a driver running in process context. There is no problem with this mode of operation, but it is possible that the networking software interrupt which performs packet processing may be delayed, reducing performance. To avoid this problems, drivers handing packets to the kernel outside of interrupt context should use: 

```
int netif_rx_ni(struct sk_buff *skb);
```

instead of `netif_rx()`. 

### Other 2.5 features

A number of other networking features were added in 2.5. Here is a quick summary of developments that driver developers may want to be aware of. 

  * **Ethtool support**. Ethtool is a utility which can perform detailed configuration of network interfaces; it can be found on [the gkernel SourceForge page](<http://sourceforge.net/projects/gkernel>). This tool can be used to query network information, tweak detailed operating parameters, control message logging, and more. Supporting ethtool requires implementing the `SIOCETHTOOL` `ioctl()` command, along with (parts of, at least) the lengthy set of ethtool commands. See `<linux/ethtool.h>` for a list of things that can be done. Implementing the message logging control features requires checking the logging settings before each `printk()` call; there is a set of convenience macros in `<linux/netdevice.h>` which make that checking a little easier. 

  * **VLAN support**. The 2.5 kernel has support for 802.1q VLAN interfaces; this support has also been working its way into 2.4, with the core being merged in 2.4.14. See [this page](<http://www.candelatech.com/~greear/vlan.html>) for information on the Linux 802.1q implementation. 

  * **TCP segmentation offloading**. The TSO feature can improve performance by offloading some TCP segmentation work to the adaptor and cutting back slightly on bus bandwidth. TSO is an advanced feature that can be tricky to implement with good performance; see the `tg3` or `e1000` drivers for examples of how it's done.
