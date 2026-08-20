---
title: The cpuidle subsystem
url: https://lwn.net/Articles/384146/
date: "April 26, 2010"
category: "ACPI; Power management-cpuidle"
author: "By Jonathan Corbet April 26, 2010"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
April 26, 2010

Your editor recently had cause to dig around in the cpuidle subsystem. It never makes sense to let such work go to only a single purpose when it could be applied toward the creation of a kernel-page article. So, what follows is a multi-level discussion of cpuidle, what it's for, and how it works. Doing nothing, it turns out, is more complicated than one might think. 

On most systems, the processor is idle much of the time. We can't always be running CPU-intensive work like kernel builds, video transcoding, weather modeling, or yum. When there is nothing left to do, the processor will go into the idle state to wait until it is needed again. Once upon a time, on many systems, the "idle state" was literally a thread running at the lowest possible priority which would execute an infinite loop until the system found something better to do. Killing the idle process was a good way to panic a VAX/VMS machine, which had no clue of how to do nothing without a task dedicated to that purpose. 

Running a busy-wait loop requires power; contemporary concerns have led us to the conclusion that expending large amounts of power toward the accomplishment of nothing is rarely a good idea. So CPU designers have developed ways for the processor to go into a lower-power state when there is nothing for it to do. Typically, when put into this state, the CPU will stop clocks and power down part or all of its circuitry until the next interrupt arrives. That results in the production of far more nothing per watt than busy-waiting. 

In fact, most CPUs have multiple ways of doing nothing more efficiently. These idle modes, which go by names like "C states," vary in the amount of power saved, but also in the amount of ancillary information which may be lost and the amount of time required to get back into a fully-functional mode. On your editor's laptop, there are three idle states with the following characteristics: 

> | C1| C2| C3  
> ---|---|---|---  
> Exit latency (µs)| 1 | 1 | 57  
> Power consumption (mW)| 1000 | 500| 100  
  
On a typical processor, C1 will just turn off the processor clock, while C2 turns off other clocks in the system and C3 will actively power down parts of the CPU. On such a system, it would make sense to spend as much time as possible in the C3 state; indeed, while this sentence is being typed, the system is in C3 about 97% of the time. One might have thought that emacs could do a better job of hogging the CPU, but even emacs is no challenge for modern processors. The C1 state is not used at all, while a small amount of time is spent in C2. 

One might wonder why the system bothers with anything but C3 at all; why not insist on the most nothing for the buck? The answer, of course, is that C3 has a cost. The 57µs exit latency means that the system must commit to doing nothing for a fair while. Bringing the processor back up also consumes power in its own right, and the ancillary costs - the C3 state might cause the flushing of the L2 cache - also hurt. So it's only worth going into C3 if the power savings will be real and if the system knows that it will not have to respond to anything with less than 57µs latency. If those conditions do not hold, it makes more sense to use a different idle state. Making that decision is the cpuidle subsystem's job. 

Every processor has different idle-state characteristics and different actions are required to enter and leave those states. The cpuidle code abstracts that complexity into a separate driver layer; the drivers themselves are often found in architecture-specific or ACPI code. On the other hand, the decision as to which idle state makes sense in a given situation is very much a policy issue. The cpuidle "governors" interface allows the implementation of different policies for different needs. We'll take a look at both layers. 

#### cpuidle drivers

At the highest level, the cpuidle driver interface is quite simple. It starts by registering the driver with the subsystem: 

```
#include <linux/cpuidle.h>
    
        struct cpuidle_driver {
    	char			name[CPUIDLE_NAME_LEN];
    	struct module 		*owner;
        };
    
        int cpuidle_register_driver(struct cpuidle_driver *drv);
```

About all this accomplishes is making the driver name available in sysfs. The cpuidle core also will enforce the requirement that only one cpuidle driver exist in the system at any given time. 

Once the driver exists, though, it can register a cpuidle "device" for each CPU in the system - it is possible for different processors to have completely different setups, though your editor suspects that tends not to happen in real-world systems. The first step is to describe the processor idle states which are available for use: 

```
struct cpuidle_state {
    	char		name[CPUIDLE_NAME_LEN];
    	char		desc[CPUIDLE_DESC_LEN];
    	void		*driver_data;
    
    	unsigned int	flags;
    	unsigned int	exit_latency; /* in US */
    	unsigned int	power_usage; /* in mW */
    	unsigned int	target_residency; /* in US */
    
    	unsigned long long	usage;
    	unsigned long long	time; /* in US */
    
    	int (*enter)	(struct cpuidle_device *dev,
    			 struct cpuidle_state *state);
        };
```

The `name` and `desc` fields describe the state; they will show up in sysfs eventually. `driver_data` is there for the driver's private use. The next four fields, starting with `flags`, describe the characteristics of this sleep state. Possible `flags` values are: 

  * `CPUIDLE_FLAG_TIME_VALID` should be set if it is possible to accurately measure the amount of time spent in this particular idle state. 

  * `CPUIDLE_FLAG_CHECK_BM` indicates that this state is not compatible with bus-mastering DMA activity. Deep sleeps will, among other things, disable the bus cycle snooping hardware, meaning that processor-local caches may fail to be updated in response to DMA. That can lead to data corruption problems. 

  * `CPUIDLE_FLAG_POLL` says that this state causes no latency, but also fails to save any power. 

  * `CPUIDLE_FLAG_SHALLOW` indicates a "shallow" sleep state with low latency and minimal power savings. 

  * `CPUIDLE_FLAG_BALANCED` is for intermediate states with some latency and moderate power savings. 

  * `CPUIDLE_FLAG_DEEP` marks deep sleep states with high latency and high power savings. 

The depth of the sleep state is also described by the remaining fields: `exit_latency` says how long it takes to get back to a fully functional state, `power_usage` is the amount of power consumed by the CPU when it is in this state, and `target_residency` is the minimum amount of time the processor should spend in this state to make the transition worth the effort. 

The `enter()` function will be called when the current governor decides to put the CPU into the given `state`; it will be described more fully below. The number of times the state has been entered will be kept in `usage`, while `time` records the amount of time spent in this state. 

The cpuidle driver should fill in an appropriate set of states in a `cpuidle_device` structure for each CPU: 

```
struct cpuidle_device {
    	unsigned int		cpu;
    
    	int			last_residency;
    	int			state_count;
    	struct cpuidle_state	states[CPUIDLE_STATE_MAX];
    	struct cpuidle_state	*last_state;
    
    	void			*governor_data;
    	struct cpuidle_state	*safe_state;
    	/* Others omitted */
        };
```

The driver should set `state_count` to the number of valid states and `cpu` to the number of the CPU described by this device. The `safe_state` field points to the deepest sleep which is safe to enter while DMA is active elsewhere in the system. The device should be registered with: 

```
int cpuidle_register_device(struct cpuidle_device *dev);
```

The return value is, as usual, zero on success or a negative error code. 

The only other thing that the driver needs to do is to actually implement the state transitions. As we saw above, that is done through the `enter()` function associated with each state: 

```
int (*enter)(struct cpuidle_device *dev, struct cpuidle_state *state);
```

A call to `enter()` is a request from the current governor to put the CPU associated with `dev` into the given `state`. Note that `enter()` is free to choose a different state if there is a good reason to do so, but it should store the actual state used in the device's `last_state` field. If the requested state has the `CPUIDLE_FLAG_CHECK_BM` flag set, and there is bus-mastering DMA active in the system, a transition to the indicated `safe_state` should be made instead. The return value from `enter()` should be the amount of time actually spent in the sleep state, expressed in microseconds. 

If the driver needs to temporary put a hold on cpuidle activity, it can call: 

```
void cpuidle_pause_and_lock(void);
        void cpuidle_resume_and_unlock(void);
```

Note that `cpuidle_pause_and_lock()` blocks cpuidle activity for all CPUs in the system. It also acquires a mutex which is held until `cpuidle_resume_and_unlock()` is called, so it should not be used for long periods of time. 

Power management for a specific CPU can be controlled with: 

```
int cpuidle_enable_device(struct cpuidle_device *dev);
        void cpuidle_disable_device(struct cpuidle_device *dev);
```

These functions can only be called with cpuidle as a whole paused, so one must call `cpuidle_pause_and_lock()` first. 

#### cpuidle governors

Governors implement the policy side of cpuidle. The kernel allows the existence of multiple governors at any given time, though only one will be in control of a given CPU at any time. Governor code begins by filling in a `cpuidle_governor` structure: 

```
struct cpuidle_governor {
    	char			name[CPUIDLE_NAME_LEN];
    	unsigned int		rating;
    
    	int  (*enable)		(struct cpuidle_device *dev);
    	void (*disable)		(struct cpuidle_device *dev);
    	int  (*select)		(struct cpuidle_device *dev);
    	void (*reflect)		(struct cpuidle_device *dev);
    
    	struct module 		*owner;
    	/* ... */
        };
```

The `name` identifies the governor to user space, while `rating` is the governor's idea of how useful it is. By default, the kernel will use the governor with the highest rating value, but the system administrator can override that choice. 

There are four callbacks provided by governors. The first two, `enable()` and `disable()`, are called when the governor is enabled for use or removed from use. Both functions are optional; if the governor does not need to know about these events, it need not supply these functions. 

The `select()` function, instead, is mandatory; it is called whenever the CPU has nothing to do and wishes the governor to pick the optimal way of getting that nothing done. This function is where the governor can apply its heuristics, look at upcoming timer events, and generally try to decide how long the sleep can be expected to last and which idle state makes the most sense. The return value should be the integer index of the target state (in the `dev->states` array). 

When making its decision, the governor should pay attention to the current latency requirements expressed by other code in the system. The mechanism for the registration of these requirements is the "pm_qos" subsystem. A number of quality-of-service requirements can be registered with this system, but the one most relevant for cpuidle governors is the CPU latency requirement. That information can be obtained with: 

```
#include <linux/pm_qos_params.h>
    
        int max_latency = pm_qos_requirement(PM_QOS_CPU_DMA_LATENCY);
```

On some systems, an overly-deep sleep state can wreak havoc with DMA operations (trust your editor's experience on this), so it's important to respect the latency requirements given by drivers. 

Finally, the `reflect()` function will be called when the CPU exits the sleep state; the governor can use the resulting timing information to reach conclusions on how good its decision was. 

#### An aside: blocking deep sleep

For what it's worth, driver developers can use these pm_qos functions to specify latency requirements: 

```
#include <linux/pm_qos_params.h>
    
        int pm_qos_add_requirement(int qos, char *name, s32 value);
        int pm_qos_update_requirement(int qos, char *name, s32 new_value);
        void pm_qos_remove_requirement(int qos, char *name);
```

This API is not heavily used in current kernels; most of the real uses would appear to be drivers telling the system that transitions into deep sleep states would be unwelcome. Needless to say, a driver should only block deep sleep when it is strictly necessary; the latency requirement should be removed when I/O is not in progress. 

And that describes the 2.6.34 version of the cpuidle subsystem and API. For the curious, the core and governor code can be found in `drivers/cpuidle`, while cpuidle drivers live in `drivers/acpi/processor_idle.c` and a handful of ARM subarchitecture implementations. All told, it's a testament to the complexity of doing nothing properly on contemporary systems.
