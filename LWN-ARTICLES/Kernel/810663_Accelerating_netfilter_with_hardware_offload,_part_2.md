---
title: "Accelerating netfilter with hardware offload, part 2"
url: https://lwn.net/Articles/810663/
date: "January 31, 2020"
category: "Device drivers-Network drivers; Networking-Packet filtering; Packet filtering"
author: "January 31, 2020 This article was contributed by Marta Rybczyńska"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

January 31, 2020

This article was contributed by Marta Rybczyńska

As network interfaces get faster, the amount of CPU time available to process each packet becomes correspondingly smaller. The good news is that many tasks, including packet filtering, can be offloaded to the hardware itself. The bad news is that the Linux kernel required quite a bit of work to be able to take advantage of that capability. The [first article](<https://lwn.net/Articles/809333/>) in this series provided an overview of how hardware-based packet filtering can work and the support for this feature that already existed in the kernel. This series now concludes with a detailed look at how offloaded packet filtering works in the netfilter subsystem and how administrators can make use of it.

The offload capability was added by [a patch set](<https://lwn.net/ml/netdev/20190709205550.3160-1-pablo%40netfilter.org/>) from Pablo Neira Ayuso, merged in the kernel 5.3 release and updated thereafter. The goal of the patch set was to add support for offloading a subset of the netfilter rules in a typical configuration, thus bypassing the kernel's generic packet-handling code for packets filtered by the offloaded rules. It is not currently possible to offload all of the rules, as that would require additional support from the underlying hardware and in the netfilter code. The use case and some of the internals are mentioned in Neira's [slides [PDF]](<https://linuxplumbersconf.org/event/4/contributions/463/attachments/286/485/2019-plumbers-lisboa.pdf>) from the 2019 Linux Plumbers Conference. 

#### Background work

The bulk of the patch set is the refactoring needed to allow the netfilter offload mechanism to reuse the infrastructure that was directly tied to the traffic-control (tc) subsystem before. The refactoring effort was able to take advantage of an existing driver callback. Some modules, which were only used by the tc subsystem before have become more generic.

The first new subsystem, the "flow block" infrastructure, [was introduced](<https://lore.kernel.org/netdev/20171012171823.1431-1-jiri@resnulli.us/>) in 2017 to allow the sharing of filtering rules and to optimize the use of [ternary content-addressable memory (TCAM)](<https://en.wikipedia.org/wiki/Content-addressable_memory>) entries. It allows a set of rules to be shared by two (or more) network interfaces, which reduces the hardware resources needed by rule offloading; this is because the network cards with multiple physical interfaces usually share the TCAM entries between those interfaces. This optimization, in the case of switches, allows the administrator to define common blocks of filtering rules that can be assigned to multiple interfaces. When a shared block is in place, any changes will apply to all interfaces that the block is assigned to. The netfilter offload patch extends the use of flow blocks beyond the tc subsystem, making it available for all subsystems that need to offload packet-filtering tasks.

A flow block is, at its core, a list of driver callbacks invoked when the rules programmed into the hardware are changed. There is usually one entry per device (for typical network cards); in the case of switches there is one callback for all the interfaces in the switch. For a configuration with two network interfaces that share the same rules, the flow-block list contains two callbacks (one for each interface). The flow-block infrastructure does not limit the number of filtering rules.

Another important part of the patch set modifies a callback provided by network card device drivers. Those callbacks are kept in the `[struct net_device_ops](<https://elixir.bootlin.com/linux/latest/source/include/linux/netdevice.h#L1257>)` structure. The netfilter offload patch set reuses the `ndo_setup_tc()` callback, which was initially added to configure schedulers, classifiers, and actions for the tc subsystem; it has the following prototype:

```
int (*ndo_setup_tc)(struct net_device *dev, enum tc_setup_type type,
                            void *type_data);
```

It takes the network device `dev`, the type of the configuration to apply (defined in the `enum tc_setup_type`) and an opaque data value. The enum defines different action types; netfilter does not define its own type, instead it uses the one defined by the flower classifier (`TC_SETUP_CLSFLOWER`). This is expected to change in the future, when drivers will start supporting tc and netfilter offloading at the same time.

Finally, the flow-rule API [was introduced](<https://lwn.net/ml/netdev/20190202115054.4880-1-pablo@netfilter.org/>) in February 2019 (there is a longer cover letter in [version 6 of the flow-rule patch set](<https://lwn.net/Articles/775046/>)). It implements an intermediate representation for the flow-filtering rules, allowing the separation of the driver-specific implementation from the details of the subsystem calling it. In particular, it enabled a single code path to be used by drivers to support access-control-list offloads configured by either `ethtool` or the flower classifier.

In the flow-rule API, each [`flow_rule`](<https://elixir.bootlin.com/linux/latest/source/include/net/flow_offload.h#L243>) object represents a filtering rule. It consists of the match condition of the rule ([`struct flow_match`](<https://elixir.bootlin.com/linux/latest/source/include/net/flow_offload.h#L9>)) and the actions to be performed ([`struct flow_action`](<https://elixir.bootlin.com/linux/latest/source/include/net/flow_offload.h#L219>)). In the netfilter code, each `flow_rule` represents a rule to be offloaded to the hardware; it is kept in the flow-block list. When netfilter offloads a rule to hardware, it iterates over the callback list in the flow block, invoking each callback and passing in the rules, so that they can be handled by the driver.

#### Driver API changes

As the tc-specific code was made more generic, several types and definitions were renamed or reorganized. A new type, `flow_block_command`, that defines the commands for the driver's flow-block setup function was added. It includes two definitions, `TC_BLOCK_BIND` and `TC_BLOCK_UNBIND`, that were renamed to `FLOW_BLOCK_BIND` and `FLOW_BLOCK_UNBIND`, respectively. Those allow the kernel to bind and unbind a flow block to an interface. In the same way, `flow_block_binder_type`, which defines the type of the offload (ingress for input and egress for output), had seen its members renamed from `TCF_BLOCK_BINDER_TYPE_*` to `FLOW_BLOCK_BINDER_TYPE_*`

The existing drivers were all setting up tc offloading in a very similar way, so Neira added a helper function that can be used by all of them:

```
int flow_block_cb_setup_simple(struct flow_block_offload *f,
            			   struct list_head *driver_block_list,
            			   flow_setup_cb_t *cb, void *cb_ident,
    				   void *cb_priv, bool ingress_only);
```

where `f` is the offload context, `driver_block_list` is the list of flow blocks for the specific driver, `cb` is the driver's `ndo_setup_tc()` callback, `cb_ident` is the identification of the context, `cb_priv` is the context to be passed to `cb` (in most cases `cb_ident` and `cb_priv` are identical), and `ingress_only` is true if the offload should be set up for the ingress (receive) side only (this was the case for all the drivers right until 5.4, in 5.5 the `cxgb4` driver supports both directions). `flow_block_cb_setup_simple()` registers one callback per network device, which is exactly what most of the drivers need. 

Each driver is expected to keep a list of flow blocks with their callbacks: that is the `driver_block_list` argument of `flow_block_cb_setup_simple()`. This list is necessary if the driver needs more than one callback, for example one for the ingress and the other for the egress rules.

The callback implemented by the drivers, of type `flow_setup_cb_t` has the following definition:

```
typedef int flow_setup_cb_t(enum tc_setup_type type, void *type_data,
            			void *cb_priv);
```

Its implementation in the driver sets up the hardware filtering using the provided configuration. The argument `type` defines the classifier to use, `type_data` is the data specific to the classifier (and is usually a pointer to a `flow_rule` structure) and `cb_priv` is the callback private data.

If the driver needs to go beyond the functionality of `flow_block_cb_setup_simple()` (which usually means it is part of a switch), it needs to use the part of the API that allocates the flow blocks directly. These blocks are allocated and freed by two helpers: `flow_block_cb_alloc()` and `flow_block_cb_free()` with the following prototypes: 

```
struct flow_block_cb *flow_block_cb_alloc(flow_setup_cb_t *cb,
                                                  void *cb_ident, void *cb_priv,
                                                  void (*release)(void *cb_priv));
        void flow_block_cb_free(struct flow_block_cb *block_cb);
```

The callbacks are defined by the drivers and passed to netfilter by the flow-block infrastructure. Netfilter maintains the list of callbacks that are attached to each given rule.

Each of the flow blocks contains a list of driver offload callbacks. The drivers can add and remove themselves from the list contained in the flow-block list using `flow_block_cb_add()` and `flow_block_cb_remove()` with the following prototypes:

```
void flow_block_cb_add(struct flow_block_cb *block_cb,
                               struct flow_block_offload *offload);
        void flow_block_cb_remove(struct flow_block_cb *block_cb,
                                  struct flow_block_offload *offload);
```

The driver can look up for a specific callback using `flow_block_cb_lookup()` defined as follows:

```
struct flow_block_cb *flow_block_cb_lookup(struct flow_block *block,
            				       flow_setup_cb_t *cb, void *cb_ident);
```

This function searches for the flow-block callbacks on the list in the `block` context; if both the `cb` callback and the `cb_ident` value match, it returns the associated flow-block callback structure. It is used by switch drivers to check if a given callback is already installed (again, switches use one callback for all of their interfaces). The setup of the first interface allocates and registers the callback when `flow_block_cb_lookup()` returns NULL. Subsequently, other interfaces get a non-NULL return and reuse the callback in place, only increasing the reference count (see below). When unregistering a callback, `flow_block_cb_lookup()` also returns non-NULL if other users exist and the driver just decrements the reference count.

The operations for the flow-block reference counts are `flow_block_cb_incref()` and `flow_block_cb_decref()`; they are defined as follows: 

```
void flow_block_cb_incref(struct flow_block_cb *block_cb);
        unsigned int flow_block_cb_decref(struct flow_block_cb *block_cb);
```

The value returned by `flow_block_cb_decref` is the value of the reference count after the operation.

Another function, `flow_block_cb_priv()`, allows the driver to access its private data. It has the following, simple, prototype:

```
void *flow_block_cb_priv(struct flow_block_cb *block_cb);
```

Finally, the drivers can use `flow_block_is_busy()` to check if the callback is already in use (added to the lists and active). The function has the following prototype:

```
bool flow_block_cb_is_busy(flow_setup_cb_t *cb, void *cb_ident,
                                   struct list_head *driver_block_list);
```

It returns `true` if it finds an entry with both `cb` and `cb_ident` on the `driver_block_list`. Its use is in the code setting up the offloads to avoid setting up tc and netfilter callbacks at the same time. This check is expected to be removed from drivers that are able to support both at the same time in their hardware, once that support gets implemented.

The internals of the traffic classifier were modified to apply the filtering stored in the flow-block API; this is done in a new function `[tcf_block_setup()](<https://elixir.bootlin.com/linux/latest/source/net/sched/cls_api.c#L1537>)`. 

#### Callback list

The drivers set up the flow-block object (`flow_block_cb`) and add their callbacks to their list. Each driver then passes this list to the core networking code, which does the registration (in tc and netfilter) and calls the driver callback to do the actual hardware setup. This callback uses the classifier-specific data it receives in the parameters, including the type of the operation (for example to add or remove an offload).

Edward Cree [asked](<https://lwn.net/ml/netdev/75eec70e-60de-e33b-aea0-be595ca625f4@solarflare.com/>) why there is a single list per driver, and not per device, for example:

Pablo, can you explain (because this commit message doesn't) why these per-driver lists are needed, and what the information/state is that has module (rather than, say, netdevice) scope? 

The drivers only supported one single flow block, Neira [explained](<https://lwn.net/ml/netdev/20190813195126.ilwtoljk2csco73m@salvia/>), and the idea was to extend that support to one for each subsystem (ethtool, tc, and so on). This is for two reasons: the first is that the current drivers can only support one subsystem; when that restriction is lifted, the other limitation is that the sharing support would require the same configuration of all the subsystems. This means that, for example, the same configuration would be required for both `eth0` and `eth1` for tc, and also there would also have to be a shared configuration for netfilter. Neira assumes this is almost never going to be the case.

#### The netfilter offload itself

[The last patch in the series](<https://lwn.net/ml/netdev/20190709205550.3160-13-pablo@netfilter.org/>) introduces the hardware offloading of netfilter itself. Currently the support is basic and only handles the ingress chain. The rule must perform an exact match on the five elements identifying the flow: the protocol, the source and destination addresses, and the source and destination ports.

An example of the offload is given in the series:

```
table netdev filter {
            chain ingress {
                type filter hook ingress device eth0 priority 0; flags offload;
                ip daddr 192.168.0.10 tcp dport 22 drop
            }
        }
```

It drops all TCP packets to the destination address `192.168.0.10`, port 22 (typically used by SSH). The only difference from the non-offloaded rules is the addition of the `flags offload` option.

Since the control of offloading is given to the administrator, there might be misconfigurations. For example, when the offload flag is set for a rule that cannot be offloaded, the error code will be `EOPNOTSUPP`. If the driver cannot handle the command, for example when the TCAM is full, the result will be a driver-specific error code.

The interface gives a lot of power to the system administrator, but also makes them responsible for figuring out which rules will benefit the most from offloading. It seems that knowledge of the system configuration and the traffic it handles will be necessary to derive the most benefit from this new feature. At the time of writing this article, no benchmark or best-practices documents are available. It also remains to be seen where the limitations of the offload feature will be -- for example, how easy it will be to diagnose failures in the user configuration coming from the driver callbacks.

#### Summary

The netfilter classification offloading feature allows the activation of hardware offloading, which can provide important performance gains for certain use cases. This work resulted in useful refactoring of existing code blocks and opens a way for other offloading users. However, drivers need to be modified to take full advantage of this capability and the API itself is quite complex with a number of levels of callbacks. The administrators gain a powerful tool, but it will be up to them to use it correctly. There is definitely more work to be done in this area. 

[The author would like to thank Pablo Neira Ayuso for helpful comments]
