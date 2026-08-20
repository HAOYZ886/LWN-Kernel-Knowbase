---
title: Active state management of power domains
url: https://lwn.net/Articles/744047/
date: "January 15, 2018"
category: Power management
author: "January 15, 2018 This article was contributed by Viresh Kumar"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

January 15, 2018

This article was contributed by [Viresh Kumar](<mailto:viresh.kumar@linaro.org>)

The Linux kernel's generic power domain (genpd) subsystem has been extended to support active state management of the power domains in the 4.15 development cycle. Power domains were traditionally used to enable or disable power to a region of a system on chip (SoC) but, with the recent updates, they can control the clock rate or amount of power supplied to that region as well. These changes improve the kernel's ability to run the system's hardware at the optimal power level for the current workload. 

SoCs have become increasingly complex and power-efficient over the years. Most of the IP blocks in an SoC have independent power-control logic that can be turned on or off to reduce the power they consume. But there is also a significant amount of static current leakage that can't be controlled using the IP-block-specific power logic. SoCs are normally divided into several regions depending on which IP blocks are generally used together, so that an unused region can be completely powered off to eliminate this leakage. These regions of the chip, called "power domains", can be present in a hierarchy and thus can be nested; a nested domain is called a subdomain of the master domain. Powering down a power domain results in disabling all the IP blocks and subdomains controlled by the domain and also stopping any static leakage in that region of the chip.

The Linux kernel's generic power domains are used to group devices that share clock or other power resources and are all enabled or disabled together, though these devices may further have fine-grained control over individual resources. Generic power domains support a limited number of operations today, most of which eventually come down to enabling or disabling the power domain to avoid static leakage. 

Powering down a power domain can have a penalty though, as powering it back up later may take a significant amount of time. Additionally, the power domain controller registers are often only accessible via the SPI and I2C buses, which are quite slow. For that reason, some of the more advanced SoCs have implemented several idle states for their power domains. A deeper idle state saves more power for the region the power domain controls, but raises the penalty to restore power to the domain. Thus it is important to avoid taking the power domain to a deeper idle state if we already know that we need to power on the domain after a short amount of time. The idle-state support was recently added to the generic power domains in the Linux kernel.

Similar to idle states, some advanced SoCs have implemented various active states for power domains. The active states control the clock rate, voltage, or power that the power domain provides to the region it controls. These active states are called "performance states" within the Linux kernel. The higher the performance state, the higher the dynamic power consumption and the static power leakage of the region controlled by the domain.

Each device controlled by the power domain can request that the power domain be configured to a performance state that satisfies the current performance requirements of the device; the power domain will be configured to the highest performance state requested by all of its devices. The performance states (within the genpd core) are identified by positive integer values; a lower value represents a lower performance state. The performance state zero is special; devices can request this state if they do not want to be considered when the next performance state for the power domain is calculated.

Linux doesn't enforce a policy on what the values of the performance states should be. Platforms can choose any range of consecutive or non-consecutive values, from 1-10 or 500-550 or anything else they want. The genpd core only compares these values against each other to find the highest integer value and passes that value to the platform-specific genpd callback (described later); that callback should have knowledge about the valid performance-state ranges for that platform.

#### Internals

The genpd core provides the following helper for devices to request a performance state for their power domain:

```
int dev_pm_genpd_set_performance_state(struct device *dev,
        					   unsigned int state);
```

Here, `dev` is the pointer to the `device` structure and `state` is the requested performance state for the power domain that controls the device. This function updates the performance state constraint of the device on its domain. The genpd core then finds the new performance state for the domain based on the current requests from the various devices the domain controls, then updates the performance state, if required, of the power domain in a platform-dependent way. This happens synchronously and the performance state of the power domain is updated before this helper returns. `dev_pm_genpd_set_performance_state()` returns zero on success and an error number otherwise. The return value `-ENODEV` is special; it is returned if the power domain of the device doesn't support performance states.

On a call to `dev_pm_genpd_set_performance_state()`, the genpd core calls the `set_performance_state()` callback of the power domain if the performance state of the power domain needs to be updated. This callback must be supplied by the power-domain drivers that support performance states.

```
struct generic_pm_domain {
    	int (*set_performance_state)(struct generic_pm_domain *genpd, 
    				     unsigned int state);
            /* Many other fields... */
        };
```

Here, `genpd` is the generic power domain and `state` is the target performance state based on the requests from all the devices managed by the `genpd`. As pointed out earlier, if the domain doesn't have this callback set, the helper `dev_pm_genpd_set_performance_state()` will return `-ENODEV`.

The mechanism by which the performance state of a power domain is updated is left for the implementation and is platform dependent. For some platforms, the `set_performance_state()` callback may directly configure some regulator(s) and/or clock(s) that are managed by Linux, while in other cases the `set_performance_state()` callback may end up informing the firmware running on an external processor (not managed by Linux) about the target performance state, which eventually may program the power resources locally.

Also note that, in the current implementation, performance-state updates aren't propagated to the master domains from the subdomains and only devices (i.e. no subdomains) directly controlled by the power domain are considered while finding its effective performance state. The reason being that none of the current hardware designs have a configuration that would need this feature; more thought needs to be put into that for various reasons. For example, there may not be a one-to-one mapping between the performance states of subdomains and those of their master domains. We can also have multiple master domains for a subdomain and the master domains may need to be configured to different performance states for a single performance state of the subdomain.

#### Interaction with the OPP layer

We have discussed how a device requests a performance-state change and how that happens internally in the genpd core, but we haven't discussed how the device drivers know which performance state to request based on their own performance requirements. Ideally, this information should come from the device tree (DT) but, after several rounds of [discussions](<https://marc.info/?l=linux-kernel&m=149410710629056&w=2>) on the linux-kernel mailing list, it was decided to merge a non-DT solution first and then attempt to add DT bindings for the power-domain performance states later. The DT bindings are being [reviewed](<https://marc.info/?l=linux-kernel&m=150945404818511&w=2>) currently on the mailing list.

The devices with power-domain performance-state requirements fall broadly into two categories:

  * Devices with fixed performance requirements that will always request the same performance state for their power domain. Drivers of such devices can hard-code the performance-state requirement in the driver or its platform data until the time that DT bindings are in place. Devices with fixed performance-state requirements can call `dev_pm_genpd_set_performance_state()` just once, when they are enabled by their drivers, and they don't need to worry about power-domain performance states after that, as genpd will always consider them while reevaluating power domain's performance state.

  * Devices with varying performance requirements, based on their own operating performance state. An example of such a device would be a Multi-Media Card (MMC) controller or a CPU. The rest of this section discusses such devices.

The discrete tuples, consisting of frequency and voltage pairs, that the device supports are called "operating performance points" (OPPs). These were explained in detail in [this article](<https://lwn.net/Articles/718632/>). 

Devices can have different performance-state requirements than their power domain, based on which OPP the devices are currently configured for. For example, a device may need performance state three for running at 800MHz and performance state seven to run at 1.2GHz. These devices would need to call `dev_pm_genpd_set_performance_state()` whenever they change their OPP if the performance state of the previous OPP is different than the new OPP.

The OPP core has been enhanced to store a performance state corresponding to each OPP of the device and can do the conversion from an OPP to device's power domain's performance state. The OPP core helper `dev_pm_opp_set_rate()` has also been updated to handle performance-state updates automatically along with clock and regulator updates.

In the absence of DT bindings to get the performance state corresponding to each OPP of the device, the OPP core has gained a pair of new helpers to link a device's OPPs to its power domain's performance states. Note that these helpers have been added temporarily to the OPP core to support initial platforms that need to configure the performance states of power domains. These helpers will be removed once the proposed DT bindings (and corresponding kernel code) are merged.

```
struct opp_table *dev_pm_opp_register_get_pstate_helper(struct device *dev,
        		     int (*get_pstate)(struct device *dev, unsigned long rate));
```

Here, `dev` is the pointer to the `device` structure and `get_pstate()` is the platform-specific callback that returns the performance state corresponding to device's `rate` on success or an error number on failure. `dev_pm_opp_register_get_pstate_helper()` returns a pointer to the OPP table on success and an error number (cast as a pointer) on failure. It must be called before any OPPs are added for the device, as the OPP core invokes this callback when OPPs are added to get the performance state corresponding to those OPPs (and hence target frequencies). `dev_pm_opp_unregister_get_pstate_helper()` takes a reference of the OPP table and that must be returned (so that the table can be freed once we don't need it anymore) with the help of the following function:

```
void dev_pm_opp_unregister_get_pstate_helper(struct opp_table *opp_table);
```

Here, `opp_table` is the pointer to the OPP table, earlier returned by `dev_pm_opp_register_get_pstate_helper()`.

The basic infrastructure is in place now to implement platform-specific power-domain drivers that allow configuring performance states. If you want to implement performance states for your power domains, then all you need to do is:

  * Implement a power domain driver (which you would do anyway, with or without performance states).
  * Implement `set_performance_state()` callback for the power domain.
  * Call `dev_pm_opp_register_get_pstate_helper()` from platform-specific code and register your helper routine that can convert device OPPs to performance states. Note that this step is only required for devices that have OPPs of their own. 
  * Hard-code the performance-state requirements in platform data or drivers for devices that do not have changing performance state requirements.

The DT bindings proposal is already under review, and code updates will be sent once the DT bindings are merged. In the future, we may also want to drive the devices controlled by a power domain at the highest OPP permitted by the current performance state of the power domain. For example, a device may have requested performance state five as it needs to run at 900MHz currently but, because of the votes from other devices (controlled by the same power domain), the effective performance state selected is eight. At this point it may be better, power and performance wise, to run the device at 1.3GHz (the highest device OPP supported at performance state eight) as that may not result in much more power consumption as the power domain is already configured for state eight. More thought is needed in this area, though.
