---
title: "Linux power management: The documentation I wanted to read"
url: https://lwn.net/Articles/505683/
date: "July 10, 2012"
category: Power management
author: "July 10, 2012 This article was contributed by Neil Brown"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

July 10, 2012

This article was contributed by Neil Brown

[Last week](<https://lwn.net/Articles/504814/>) we discussed three elements that might serve to guide the creation of introductory technical documentation. This week we put those elements to the test by using them to create some introductory documentation for Linux power management. For me, this exercise precisely answers the question "What were you looking for that you didn't find?", as it is the documentation I would have liked to read. 

This documentation is necessarily incomplete, partly because my own experience is not yet broad enough to provide a comprehensive document, and partly because doing so might try the patience of the present readership. As such it stops short of delving into the details of hibernation and completely omits any treatment of quality-of-service and wakeup sources, all of which would have an important place in a more complete document. Fortunately there are still sufficient topics to showcase the presentation of structure, purpose, and examples. 

#### Three perspectives on Linux power management

The power management infrastructure in Linux is quite complex, but hopefully not intractably so. To get a handle on this complexity it is helpful to view it from three different perspectives. The first perspective highlights the different holistic states of the system which roughly divide into "in use", "not in use", and "indefinitely not in use", corresponding to "run time power management", "suspend" and "hibernate". One of the distinctions between these is the size of the power switch. The first uses lots of little power switches at different times, while the last turns off everything all at once (except maybe a real-time clock or similar). 

The second of these states is somewhat harder to define. It covers a range of states which are not easy to clearly differentiate. At one end of the spectrum we have the traditional "suspend" mode of a laptop, which is almost like hibernation but uses a little more power and is a little quicker to get into and out of. Once the laptop has entered suspend it really must stay there using minimal power until it is explicitly wakened, as it might have been placed in a padded case for transport and any increase in power usage could result in over-heating and damage. This state is often entered with help from BIOS firmware so, to the OS, it is a bit like a single power switch which transitions from "on" to "suspend". 

At the other end of the spectrum is the way that "suspend" is used in the Android mobile platform and similar devices. These devices are expected to wake up spontaneously for various reasons, whether due to an incoming phone call, a reminder alarm, or just a periodic check for new email or software updates. Management of power and temperature is generally better than notebooks so the risk of over-heating is not present. There is normally little or no firmware and the entire power-management transition is handled by the OS, so it is responsible for turning off each individual device in the correct order, and then restoring them again later. 

Between these extremes of a light hibernation and a heavy snooze there is room for other possibilities. A server might use a BIOS-based suspend to save power after arranging for wake-ups via wake-on-LAN or a realtime clock alarm. This can be seen as a deeper sleep than an Android phone normally enters, but not as deep as the laptop in its padded case. The "suspend" mode in Linux attempts to cater to all of these and that flexibility leads to some of the complexity. 

The second perspective highlights the broad variety in components that need to be managed. Some, like rotating disk drives, have a high cost in power and time for turning off and on again, while others like an LED have essentially no cost. Some, such as a UART, need to either be off or sufficiently on to be able to accept full-rate data at any moment. Others, such as USB, can enter intermediate states where they can receive external signaling, but are free to take some time to fully wake up. 

Other sources of variability include the level of independence from other devices, the degree of involvement of user-space in management of the device, and how power is routed - whether through the same bus that commands and data flow, or through some separate regulator or "power domain". These are just some of the ways that devices can vary and thus some of the issues that Linux power management needs to be prepared for. 

The final perspective highlights the different stages on the way towards a low-power state, and on the way back to full functionality. The key elements of the low-power transition are to move all relevant components to a quiescent state, to record that state, then to stop powering some or all of the components; similar elements apply on the way back up. The details of managing all the aforementioned complexity through this simple transition means that we have quite a few stages as we will shortly see. 

#### Two Understandings

Part of understanding the solution to managing this complexity is understanding the balance that has been chosen between a ["mid-layer"](<https://lwn.net/Articles/336262/>) solution and a "library" solution. That is, how much responsibility for correct behavior and sequencing is taken by the core code and imposed on the drivers, and how much of the responsibility is left in the hands of the drivers. Centralizing responsibility tends to be safe but inflexible, while distributing it is risky but versatile. Linux power management takes a middle road, so it is important to understand where each responsibility lies. 

The main imposition made by the PM core is the over-all sequencing of suspend and resume. Allowing individual drivers to take a more active role in this process would probably require a general dependency solver and would undoubtedly make debugging a lot harder. In contrast, choices that are local to a specific device, such as timeouts before power management activates, or the use of a separate thread for performing power management actions are actively supported by the core without being imposed on drivers that don't want them. 

One other imposition, which will be raised again later, involves interaction with interrupts. The PM core strongly encourages a specific sequencing, but does provide hooks for a driver to escape it if absolutely necessary. 

Understanding Linux power management also requires knowing how devices are classified in Linux. The most obvious classification is shown by the "`subsystem`" link that can be found in the `sysfs` entry for the device. This points to either a "bus" or a "class" that the device belongs to. This subsystem roughly describes the interface that the device provides. Together with this can be a "device type" which allows further specialization. A simple example is that members of the class "block" - which are block devices such a disk drives - can be of type "disk" or type "partition" reflecting the fact that both the whole device and each individual partition is a block device, but that they do have some specific behaviors that are quite different. 

Finally each device can have a "power domain" (or `pm_domain`). This is an abstraction that is currently only used for ARM SoC modules and represents the fact that different collections of devices within the SoC can be powered on or off together, thus the power domain may need to know when each device changes power state so it can re-evaluate or adjust the overall state of the domain. 

These classifications are used to direct all the power management calls that are described below. If a device has a power domain, it gets to handle the call. If not, but the type, class, or bus declares any PM operations, those operations get to handle the call, otherwise the call is handled by the driver for the device. The PM core doesn't attempt to call all of the possible handlers for a particular device, just the first that is found. This is an example of distribution of responsibility. The first handler has the freedom to call more specific handlers, or to do all the work itself, and equally has the responsibility to ensure all required handlers are called. 

For example, the power domain handler for the OMAP platform (in `arch/arm/plat-omap/omap_device.c`) calls the driver-specific handler (bypassing any subsystem handlers) before or after doing any OMAP-specific handling. The MMC bus handlers call into driver-specific handlers which are stored in a non-standard location - presumably for historical reasons. 

With these perspective and understandings in place, we can move on to some specifics. 

#### Runtime PM

Runtime power management has the fewest states and so is probably the best place to start digging into details. This is the part of Linux PM that manages power for individual devices without taking the whole system into a low-power state. 

In this case the most interesting stage of the transition to lower power is "move to quiescent state". Once that is done there is one method call (`runtime_suspend()`) which combines "record current state" and "remove power", and another (`runtime_resume()`) which must restore power and reload any needed device state. 

For runtime PM, the "move to quiescent state" transition is a cause, not an effect - the new state isn't requested, it is simply noticed. The PM core keeps track of the activity of each device using two counters and an optional timer. One counter (`usage_count`) counts active references to the device. These may be external references such as open file handles, or other devices that are making use of this one, or they may be internal references used to hold the device active for the duration of some operation. The other counter (`child_count`) counts the number of children that are active. The timer can be used to add a delay between the counters reaching zero and the device being considered to be idle. This is useful for devices with a high cost for turning on or off. 

This "autosuspend" timer is not widely used at present, with only nine drivers calling `pm_runtime_put_autosuspend()` to start the timer, while 14 call `pm_runtime_set_autosuspend_delay()` which sets the timeout (though that can be set via sysfs). One user is the `omap_hsmmc` driver for the High Speed Multi-Media Card interface in OMAP processors. It sets a 100ms delay before declaring a device to be truly idle, presumably due to costs in activating and deactivating the cards. 

The counter of active children can optionally be ignored when determining whether a device is idle. Normally the parent is needed to access the child - typically the parent is a bus sending commands to the child - so powering down the parent while children are active would be counterproductive. Sometimes it is useful though. 

One good example is an I2C bus. I2C (inter-integrated circuit) is a very simple 2-wire bus for signaling between integrated circuits on a board. It doesn't carry power, only a clock signal and a bidirectional data signal. The bus is entirely master-driven. Slaves (which appears as children in the Linux device tree) cannot signal the master directly at all, they simply respond to commands from the master. 

As an I2C controller is very cheap to turn on before a command is sent, and off after the response is received, there is no need to keep it powered just because its child (which could be a sensor that is monitoring the environment and may have a higher turn-on cost) is left on. Consequently some I2C controllers, such as `i2c-nomadik` and `i2c-sh_mobile` use `pm_suspend_ignore_children()` to allow them to report as idle even when they have active children. 

When a device is deemed to be idle by the above criteria its `runtime_idle()` method is called. This function will normally perform any further checks (as does `usb_runtime_idle()`) and possibly call `pm_runtime_suspend()` to initiate the change in power state. For a slight variation, `lnw_gpio_runtime_idle()` in the `gpio-langwell.c` driver doesn't call `pm_runtime_suspend()` directly but rather calls `pm_schedule_suspend()` with a 500ms delay. Presumably this design predates the introduction of the autosuspend feature. 

There is one class of devices that does not follow this structure for power management, and that is the CPU. The general pattern of entering a quiescent state, recording state information, and reducing power usage is the same, however the particular implementation is vastly different. This is partly due to the uniquely central role that the CPU plays, and partly due to the fact that a CPU typically has many more levels and styles of power saving. Runtime PM for the CPU is implemented using the `cpuidle`, `cpufreq`, and `CPU hotplug` subsystems, which will not be discussed further here; see [this article](<https://lwn.net/Articles/384146/>) for an introduction to cpuidle. 

#### System Suspend

It can be helpful to view the "suspend" process as forcing all devices into a quiescent state, and then simply allowing runtime power management to put them all to sleep. The last to go to sleep would be the CPU (or CPUs) under the guidance of "cpuidle". While this isn't the way it is actually implemented, it provides a perspective which exposes the relationship between suspend and runtime PM quite well. 

There are several reasons for not implementing it this way. Possibly the most unavoidable is that PM_RUNTIME and SUSPEND are separate kernel config options and there is a desire to keep it that way, so neither can rely on the other being present. There is also the fact that a BIOS (such as ACPI) might be involved in one or the other and may impose different handling requirements. Finally, individual drivers might want to make different decisions based on what sort of power management is happening, so it is generally best to tell them what is actually happening, rather than pretending that one thing is a form of another. 

Forcing devices into a quiescent state has an important difference from just allowing them to get there on their own - any interdependencies between devices need to be explicitly handled. Linux PM has chosen to manage this by having a clear sequence of steps for transitioning to low power, and an explicit ordering of devices so that they make each step in a well defined order. 

The ordering (stored in `dpm_list` linked through `dev->power.entry`) is normally the order in which devices are registered, with new devices added to the end, thus being after any devices that they depend on. However it is possible to reorder the list using `device_move()` which gives a device a new parent, and can place it directly after that parent, or at the end of the list. For example, when an `rfcomm` tty-over-bluetooth device is opened, a bluetooth connection is created and the tty device is reparented under the relevant bluetooth device and placed at the end of the device list. 

The first stage of suspend, after some preliminaries like calling "sync" to flush out dirty data and switching to a separate virtual console, is to move all processes into a quiescent state. Devices which interact closely with processes need a chance to have one last chat before their process goes to sleep and this is achieved by registering a "notifier" which gets called before processes are put to sleep, and again when they are woken up. 

This is variously used to: 

  * load firmware in case it will be needed during resume 

  * copy device state to swappable memory as may be needed when the device state can be enormous such as video RAM ([drivers/gpu/drm/vmwgfx](<http://lxr.linux.no/#linux+v3.4/drivers/gpu/drm/vmwgfx/vmwgfx_drv.c#L991>)) 

  * avoid deadlock when interacting with sysfs (`[drivers/acpi/battery.c](<http://lxr.linux.no/#linux+v3.4/drivers/acpi/battery.c#L953>)`) 

  * preemptively remove devices that might be removed while the system is suspended, so appropriate cleanup can happen ([drivers/mmc/core](<http://lxr.linux.no/#linux+v3.4/drivers/mmc/core/core.c#L2358>)) 

and a few other minor tasks. Once these notifiers run, all processes are sent a special signal which results them being moved to the "freezer" where they are forced to wait for system resume to happen. 

Once all processes are quiescent, the next step is to instruct all devices to also become quiescent. To do this we need to walk the list in reverse order, putting children to sleep before their parents \- as the parent may be needed to help put the child to sleep. However as a new child could be born at any moment (e.g. due to a device being plugged in), and as children get added to the end of the list, we might miss some children on the first pass. To avoid this, the PM core makes two passes over the list. The first pass starts at the beginning and simply asks devices to stop adding children by calling their "`prepare()`" method. If children are born during this time they can only be added after the current pointer in the list, and so will not be missed. Once this is complete we know that no new devices will be added, so the list is walked in the reverse order calling the "suspend" method. 

The "suspend" method is actually three separate methods, `suspend()`, `suspend_late()`, and `suspend_noirq()`, which can share among themselves the three tasks of making the device quiescent, saving any state, and reducing power usage. How much of which task is allocated to which methods is largely up to each driver providing that the division works with the calling patterns of the three methods. 

Calls to these methods are made to all devices in child-before-parent order and the sets of calls are interleaved with system-wide suspend operations, made largely through the `suspend_ops` dispatch table. The ordering is roughly: 

  * system wide `begin()`
  * per-device `suspend()`
  * system wide `prepare()`
  * per-device `suspend_late()`
  * system wide disable (almost) all interrupt handlers 
  * per-device `suspend_noirq()`
  * system wide `prepare_late()`
  * disable nonboot CPUs 
  * `syscore_suspend()`
  * system wide `enter()`

Note that it is possible for the sequence from system wide `prepare()` onwards to be repeated (after being unwound by corresponding "resume" actions) without going all the way up to fully awake and starting the sequence from the top. This happens if the `suspend_again()` suspend operation requests it. Currently this is only requested by the charger manager which often needs to wake up parts of the system to check battery charging state, without wanting the cost of a full wakeup. 

Deducing the purpose of these method calls by looking for example usage in the code is problematic for a number of reasons. 

  * For the system-wide operations (`begin()`, `prepare()`, `prepare_late()`), there are few users and those that exist do not make their purpose clear to an untrained observer. The most complete user is ACPI, so possibly a full understanding of that specification would help. Unfortunately that is beyond the scope of this article (and of this author). 

In general, ACPI recommends specific procedures for entering and leaving system sleep states (such as suspend) and Linux PM was modeled on that and then adjusted to meet broader needs. For example, `prepare_late()` was [added](<http://git.kernel.org/?p=linux/kernel/git/torvalds/linux.git;a=commitdiff;h=6a7c7eaf71b636f197d73b381a2ab729ebdcfb2e>) to resolve a conflict between the needs of ACPI and the needs of the ARM platform. 

  * For the per-device operations, `suspend_late()` was only recently added (commit cf579dfb82550e3) and there are no users in Linux-3.4. So any examples we find may be working around the absence of `suspend_late()` and so should not be copied. 

  * The initial reason for producing this document was finding code in drivers that simply wasn't working correctly and trying to understand what "correctly" might mean. Those drivers clearly cannot be used as good examples and there is [evidence](<http://www.mail-archive.com/linux-mmc@vger.kernel.org/msg13373.html> "Note: blatant abuse of the PM infrastructure") that other drivers aren't always doing the right thing, so any example may be equally faulty. 

Examining the documentation brings a little more useful information. 

  * `suspend()` should leave the device in a [quiescent state](<http://lxr.linux.no/#linux+v3.4/include/linux/pm.h#L108>)

  * `suspend_late()` can often be the [same](<http://lxr.linux.no/#linux+v3.4/include/linux/pm.h#L115>) as `runtime_suspend()` (see also [commit cf579dfb82550e3](<http://git.kernel.org/?p=linux/kernel/git/torvalds/linux.git;a=commit;h=cf579dfb82550e3>)) 

  * `suspend_noirq()` happens after interrupts are disabled and is useful when [shared interrupts](<http://lxr.linux.no/#linux+v3.4/Documentation/power/devices.txt#L345>) are used as you can be certain that the interrupt handler will not be called after `suspend_noirq()` runs. Some interrupts, such as the [timer interrupt](<http://lxr.linux.no/#linux+v3.4/kernel/irq/pm.c#L21>), are not disabled. 

One observation from the code that seems to be important before we try to paint the big picture is that, after calling the `suspend()` method on a device, runtime power management is [disabled](<http://lxr.linux.no/#linux+v3.4/drivers/base/power/main.c#L1096>). The purpose of this seems to be to stop runtime PM from racing with system suspend PM - we really don't want two threads trying to power off a device at once, and this is the interlock that prevents that. It also prevents runtime PM from powering the device back on again, so any device that might be needed in the late states of power management needs to be left on when runtime PM is disabled. 

Tying all these threads together we get that: 

  * The `suspend()` method should cause the device to stop doing anything, and enter a state much like it would be just before runtime PM might decide to turn it off. So it should wait for any DMA requests to complete and ensure new ones won't start. It should stop transmitting information and ensure that incoming information is either ignored, or triggers a wake-from-suspend (possibly marking the interrupt for wakeups). It should cancel any timers and generally prepare for nothing to happen for a while. 

If the device might be needed to power down other devices, such as an I2C controller that might be needed to tell some regulator to turn off, then the device should be activated for runtime PM purposes so that it will still be active when runtime PM is disabled. 

Part of the task of ignoring incoming information is to ensure that no new children will be created much like the runtime PM `prepare()` method does. Having new devices appear after `suspend()` would be awkward. 

  * The `suspend_late()` method should power off the device in much the same way that `runtime_suspend()` does, and it may be exactly the same routine as `runtime_suspend()`. Occasionally preparing the device to wake up may differ between the system suspend and runtime PM cases. This would be one situation where `suspend_late()` might need to be different from `runtime_suspend()`. 

The only case where `suspend_late()` should not be used is where interrupts might still be delivered, but the interrupt handler cannot tolerate the device being off. In many cases the `suspend()` routine will have put the device in a state in which it will not generate interrupts. Likely exceptions to this are when the interrupt line is shared, or when the device supports wake-from-suspend and so deliberately does not disable interrupts. 

If the platform that the device runs on uses BIOS support to enter suspend, then it is possible that this support will power off the device, so `suspend_late()` does not need to bother. If it doesn't, it could still be that the device gets powered off by instructing the BIOS to effect the state change, and it may require different power-off procedures for runtime PM and for entering suspend. If this is the case, then `suspend_late()` will quite likely be very different from `runtime_suspend()`. 

  * The `suspend_noirq()` method is an alternative to `suspend_late()` but is run without interrupts enabled. It is unlikely that any driver will provide both methods. 

Having interrupts disabled means not only that an interrupt will not occur at an awkward time, but also that using any functionality that requires interrupts will not work. So if the driver uses an I2C bus or similar to tell the device to turn off, and if the I2C bus uses interrupts to indicate completion (which is normal), then either the device must be powered-off in `suspend_late`, or the I2C interrupt must be marked IRQF_NO_SUSPEND. 

Paired with each of these methods is a method that is called when returning back towards full-functionality: `resume_noirq()`, `resume_early()` and `resume()`. These simply do the reverse of what the corresponding "suspend" function did. 

#### Closing

Structure, purpose, and examples - these seem to be the elements that distinguish good documentation and enable the reader not just to collect knowledge but to gain understanding. I'll leave you, dear reader, to be the judge of whether their presence here is sufficient to bring an understanding of power management, or indeed an understanding of quality documentation. 

I would like to thank Rafael Wysocki for valuable review of an early draft of this article.
