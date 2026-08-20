---
title: Better device power management for 3.2
url: https://lwn.net/Articles/466230/
date: "November 8, 2011"
category: "Power management-Device power management"
author: "By Jonathan Corbet November 8, 2011"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
November 8, 2011

The Linux kernel has long had the ability to regulate the CPU's voltage and frequency for optimal behavior, where "optimal" is a function of both performance and power consumption. But a system is more than just a CPU, and there are many other components which are able to run at multiple performance levels. It is unsurprising that a proper infrastructure for managing device operating points has lagged that for the CPU, since the amount of power to be saved is usually smaller. But now that CPU power behavior is fairly well optimized, the power infrastructure is growing to encompass the rest of the system. The 3.2 kernel will have a new set of APIs intended to allow drivers to let the system find the best operating level for the devices they manage. 

There are three separate pieces to the dynamic voltage and frequency scaling (DVFS) API, the first of which was actually merged for the 2.6.37 release. The "operating power points" module simply tracks the various operating levels available to a given device; the API is declared in `<linux/opp.h>`. Briefly, operating points are managed with: 

```
int opp_add(struct device *dev, unsigned long freq, unsigned long u_volt);
        int opp_enable(struct device *dev, unsigned long freq);
        int opp_disable(struct device *dev, unsigned long freq);
```

Operating points are enabled by default; a driver may disable specific points to reflect temperature or performance concerns. There is a set of functions for retrieving operating points above or below a given frequency, useful for moving up or down the power/performance scale. 

A driver wanting to support DVFS on a specific device would start by filling in one of these structures (declared, along with the rest of the API, in `<linux/devfreq.h>`): 

```
struct devfreq_dev_profile {
    	unsigned long initial_freq;
    	unsigned int polling_ms;
    
    	int (*target)(struct device *dev, unsigned long *freq);
    	int (*get_dev_status)(struct device *dev,
    			      struct devfreq_dev_status *stat);
    	void (*exit)(struct device *dev);
        };
```

Here `initial_freq` is, unsurprisingly, the original operating frequency of the device. Almost everything else in this structure is there to help frequency governors do their jobs. If `polling_ms` is non-zero, it tells the governor how often to poll the device to get its usage information; that polling will take the form of a call to `get_dev_status()`. That function should fill the `stat` structure with the relevant information: 

```
struct devfreq_dev_status {
    	/* both since the last measure */
    	unsigned long total_time;
    	unsigned long busy_time;
    	unsigned long current_frequency;
    	void *private_data;
        };
```

The governor will use this information to decide whether the current operating frequency should be changed or not. Should a change be needed, the `target()` callback will be called to change the operating point accordingly. This function should pick a frequency at least as high as the passed in `*freq`, then update `*freq` to reflect the actual frequency chosen. The `exit()` callback gives the driver a chance to clean things up if the DVFS layer decides to forget about the device. 

Once the `devfreq_dev_profile` structure is filled in, the driver registers it with: 

```
struct devfreq *devfreq_add_device(struct device *dev,
    				       struct devfreq_dev_profile *profile,
    				       const struct devfreq_governor *governor,
    				       void *data);
```

If need be, a driver can supply its own governor to manage frequencies, but the kernel supplies a few of its own: `devfreq_powersave` (keeps the frequency as low as possible), `devfreq_performance` (keeps the frequency as high as possible), `devfreq_userspace` (allows control of the frequency through sysfs), and `devfreq_simple_ondemand` (tries to strike a balance between performance and power consumption). 

The notifier mechanism built into the operating power points code can be used to automatically invoke the governor should the set of available power points change. There are a number of ways in which that change could come about; one of those is a change in expectations regarding how quickly the device can respond. For this case, 3.2 also gained an enhancement to the [quality-of-service (pm_qos) code](<https://lwn.net/Articles/386139/>) to handle per-device QOS requirements. Kernel code can express its QOS expectations for a device using these functions (all from `<linux/pm_qos.h>`): 

```
int dev_pm_qos_add_request(struct device *dev, struct dev_pm_qos_request *req,
    			       s32 value);
        int dev_pm_qos_update_request(struct dev_pm_qos_request *req, s32 new_value);
        int dev_pm_qos_remove_request(struct dev_pm_qos_request *req);
```

The `dev_pm_qos_request` structure is used as a handle for managing requests, but calling code does not need to access its internals. The passed `value` describes the desired quality of service; the documentation is surprisingly vague on just what the units of `value` are. It would appear to describe the desired latency, but the desired precision is unclear. 

On the driver side, the notifier interface is used: 

```
int dev_pm_qos_add_notifier(struct device *dev,
    			    	struct notifier_block *notifier);
        int dev_pm_qos_remove_notifier(struct device *dev,
    			           struct notifier_block *notifier);
```

When a device's quality-of-service requirements are changed, the notifier will be called with the new value. The driver can then adjust the available operating power points, disabling any that would render the device unable to meet the specified QOS requirement. 

It is worth noting that none of the new code has any in-tree users as of this writing. That suggests that the interface might be more than usually volatile; once developers try to make use of this facility, they are likely to find things that can be improved. But, then, internal interfaces are always subject to change; regardless of any evolution here, the underlying capability should prove useful.
