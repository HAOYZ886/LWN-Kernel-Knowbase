---
title: Reworking NAPI
url: https://lwn.net/Articles/214457/
date: "December 18, 2006"
category: "NAPI; Networking-NAPI"
author: 
---

NAPI ("new API," though it is not so new anymore) is an interrupt mitigation mechanism used with network devices. When network traffic is heavy, the kernel can safely predict that incoming packets will be available anytime it gets around to looking, so there is no need to have the adapter interrupting it (possibly thousands of times per second) to tell it about those packets. So a NAPI-compliant driver will turn off the packet receive interrupt and provide a `poll()` method to the kernel. When the kernel is ready to deal with more packets, `poll()` will be called with a maximum number of packets it is allowed to feed into the kernel; it should process up to that many packets and quit. 

With NAPI in place, the kernel can process significantly higher packet loads. The reduction in interrupt load helps, but there are a couple of other advantages as well. The way NAPI works makes it less likely that packets will be reordered in the kernel. And if traffic reaches the point where the kernel is forced to drop packets, those packets can be dumped before they are ever fed into the network stack. For more information on NAPI, see [this old LWN article](<http://lwn.net/Articles/30107/>) or [this page at OSDL](<http://linux-net.osdl.org/index.php/NAPI>), which is newer and more complete. 

That page may require some updating soon, however, as Stephen Hemminger has [proposed a newer NAPI](<http://lwn.net/Articles/214186/>) (NNAPI?) which changes the driver API somewhat. In the current mainline, there are two NAPI-related fields in the `net_device` structure: `poll()`, being the function called to collect packets from the adapter, and `weight`, which is essentially the driver writer's best guess as to how important the interface is relative to any others which might be on the system. Stephen's patch moves these parameters into a separate structure (`struct napi_struct`), aggregating them with a few other NAPI-related structures. 

The `napi_struct` structure is then put back into `struct net_device`, but drivers need not use that one. The whole purpose of this patch would appear to be to separate the NAPI-related information from specific network devices. There are some adapters which provide multiple ports, all of which have a single receive interrupt. The separated NAPI information allows all of those ports to share a single NAPI state and a single `poll()` function; this organization better fits the reality of the hardware. 

This patch won't hit mainline before 2.6.21, so authors have some time to react. The changes are relatively simple to make. The first is to find a `napi_struct` structure for the device; in the absence of a reason to do otherwise, the best solution would be to use the new `napi` field in the `net_device` structure. So, if the current code initializes itself with something like: 

```
dev->weight = MY_WEIGHT;
        dev->poll = my_poll;
```

The new version would look like this: 

```
dev->napi.weight = MY_WEIGHT;
        dev->napi.poll = my_poll;
```

The prototype of the `poll()` function has changed a bit, however; it now looks like: 

```
int (*poll)(struct napi_struct *napi, int budget);
```

The pointer to the `net_device` structure has been replaced with a pointer to the `napi_struct` structure. In most cases, the `net_device` pointer can be had with a call like: 

```
struct net_device *dev = container_of(napi, struct net_device, napi);
```

The meaning of the `budget` parameter has changed slightly as well; it is now the only indicator of how many packets the `poll()` function may feed into the kernel. There is no longer any need to check the `quota` field separately. Finally, the return value should be the number of packets which were actually processed. 

The other NAPI-related functions in the network system have been modified in fairly predictable ways. NAPI polling is started with either of: 

```
void napi_schedule(struct napi_struct *napi);
        /* or */
        int napi_schedule_prep(struct napi_struct *napi);
        void __napi_schedule(struct napi_struct *napi);
```

Polling is turned off with: 

```
void napi_complete(struct napi_struct *napi);
```

The current patch is in an early state, so the interfaces could change over the next few months. Nobody has spoken out against it, though, so chances are good that it will be merged in some form.
