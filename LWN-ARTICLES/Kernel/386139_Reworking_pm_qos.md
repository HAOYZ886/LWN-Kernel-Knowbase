---
title: Reworking pm_qos
url: https://lwn.net/Articles/386139/
date: "May 4, 2010"
category: Power management
author: "By Jonathan Corbet May 4, 2010"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
May 4, 2010

Aggressive power management is increasingly used to reduce the power requirements of our systems. Sometimes, though, power management can, through the creation of excessive latencies, get in the way of work which needs to be done. One way to avoid problems is to have latency-sensitive parts of the kernel express their requirements, which can then be taken into account by the power management code. Tracking these requirements is the task of the pm_qos ("power management quality of service") code. Chances are that pm_qos will see a significant API change in 2.6.35. 

The pm_qos code currently defines three quality of service parameters for which requirements may be specified: CPU latency (`PM_QOS_CPU_DMA_LATENCY`), network response latency (`PM_QOS_NETWORK_LATENCY`), and network throughput (`PM_QOS_NETWORK_THROUGHPUT`). The first two are specified in microseconds; throughput is specified in KB/sec. Currently, CPU latency requirements are observed by the [cpuidle](<http://lwn.net/Articles/384146/>) subsystem, and network latency is observed only by the mac80211 layer. Any requests for a minimum network throughput will fall on deaf ears in current kernels; given the effectiveness of asking your editor's ISP for better service, one assumes that the ignoring of throughput requests is simply a clever elimination of useless work by the networking hackers. 

The API for specifying quality of service parameters is: 

```
#include <linux/pm_qos_params.h>
    
        int pm_qos_add_requirement(int qos, char *name, s32 value);
        int pm_qos_update_requirement(int qos, char *name, s32 value);
        void pm_qos_remove_requirement(int qos, char *name);
```

For each of the above functions, `qos` is one of the parameters listed above, `name` identifies the subsystem specifying the requirement, and `value` is the new requirement. The `name` string is used to identify a specific request in `pm_qos_update_requirement()` and `pm_qos_remove_requirement()`; it must match the value given when the requirement was first added. 

Kernel code which may make decisions affecting quality of service should pay attention to the current requirements. There are two ways of doing that, one of which being to just ask pm_qos what the tightest requirement in effect is: 

```
int pm_qos_requirement(int qos);
```

The alternative is to register a notifier which is called whenever a given requirement changes, using: 

```
int pm_qos_add_notifier(int qos, struct notifier_block *notifier);
        int pm_qos_remove_notifier(int qos, struct notifier_block *notifier);
```

This API has been around for some time, though it remains lightly used within the kernel. One complaint which has been made is that the use of strings to identify requirements leads to inefficient behavior: changing a requirement involves walking a list and doing a bunch of string comparisons. Requirements are, by their nature, specified by latency-sensitive code, so it makes sense that the process should be fast. The use of arbitrary strings also opens up a distant possibility of confusion should two developers accidentally choose the same name. 

In response to these problems, pm_qos hacker Mark Gross has [proposed some changes](<http://lwn.net/Articles/385657/>) to the API. With the new version, "requirements" would become "requests," and the use of strings to identify them would be removed. The new API for the specification of ~~requirements~~ requests is: 

```
struct pm_qos_request_list *pm_qos_add_request(int qos, s32 value);
        void pm_qos_update_request(struct pm_qos_request_list *pm_qos_req,
    			       s32 new_value);
        void pm_qos_remove_request(struct pm_qos_request_list *pm_qos_req);
```

The `pm_qos_request_list` structure type is opaque to callers; it serves only as a handle to identify a specific request. Changes and removals can now be done with no list traversals and no string comparisons. On the other side, `pm_qos_requirement()` becomes `pm_qos_request()`, but the API is otherwise unchanged. 

This change seems uncontroversial, and it should address the criticisms which have been made against this API. Unless something surprising happens, the new API will probably be merged for 2.6.35.
