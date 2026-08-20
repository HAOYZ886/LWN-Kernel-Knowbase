---
title: The difficult task of doing nothing
url: https://lwn.net/Articles/647616/
date: "June 9, 2015"
category: Power management
author: "By Jonathan Corbet June 9, 2015 LinuxCon Japan"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
June 9, 2015

* * *

[LinuxCon Japan](<https://lwn.net/Archives/ConferenceByYear/#2015-LinuxCon_Japan>)

Kristen Accardi started her LinuxCon Japan session with the claim that idle is the most important workload on most client systems. Computers placed in offices are busy less than 25% of the time; consumer systems have even less to do. So idle performance, especially with regard to power consumption, is important. The good news is that hardware engineers have been putting a lot of work into reducing the power consumption of idle systems; the bad news is that operating systems are often failing to take full advantage of that work. 

In the "good old days," Kristen said, power management was relatively easy — and relatively ineffective. The "Advanced Power Management" (APM) mechanism was entirely controlled by the BIOS, so operating systems didn't have to pay much attention to it. Intel's "SpeedStep" offered exactly one step of CPU frequency scaling. The operating system could concern itself with panel dimming on laptop systems. That was about the extent of the power-management capabilities provided by the hardware at that time. 

With the rise of the mobile market, though, power management started to get more complicated. ACPI was introduced, putting more power-management work into the operating system's domain. With ACPI came the notion of "S-states" (for system-wide power-management states), "C-states" (for CPU idle states), and "P-states" (for performance-level states — frequency and voltage scaling). There can be up to 25 of these states. 

But things do not stop there; in recent years there has been an explosion of power-management features. They have names like SOix (a new low-power state) and PSR ("panel self refresh"). All of these features must be understood by the operating system, and all must work together for them to be effective. 

#### Degrees of idleness

There are, Kristen said, three fundamental degrees of idleness in a system, differing in the amount of power they use and the time it takes to get back to an active state. The level with the lowest power consumption is "off." That is an increasingly uninteresting state, though; many consumer devices no longer have an "off" switch at all. Operating system support for the "off" state tends to be easy, so there wasn't much to talk about there. 

The other two states are "suspend" and "runtime idle". A suspended system is in an intermediate state between running and off; runtime idle is closer [![\[Kristen Accardi\]](https://static.lwn.net/images/conf/2015/lcj/KristenAccardi-sm.jpg)](<https://lwn.net/Articles/647626/>) to a running system with nearly instant response when needed. The support for the two states in the kernel is entirely different in a number of ways. Suspend is a system-wide state initiated by the user, while runtime idle is a device-specific state that happens opportunistically. In a suspended system, all tasks are frozen and all devices are forced into the idle state; with runtime idle, instead, tasks are still scheduled and devices may be active. Suspend can happen at any time, while runtime idle only comes into play when a device is idle. 

Device drivers must support these two states separately; it is more work, but it's important to do. But platform-level support is also important. In current times, everything is a system-on-chip (SoC) with a great many interactions between components. If one of those components is active, it can keep the entire system out of the lower-power states. 

To see how that can come to pass, consider the "latency tolerance reporting" (LTR) mechanism built into modern buses. Any device on the bus can indicate that it may need the CPU's attention within a given maximum time (the latency tolerance). The CPU, in turn, maintains a table describing the amount of time required to return to active operation from each of its idle states. When the CPU is ready to go into a low-power state, the latency requirements declared by active devices will be compared against that table to determine the lowest state that the CPU can go into. So, if a device is running and declaring a tight latency tolerance, it can prevent the CPU from entering a deep-idle state. 

#### Where the trouble lies

Kristen then gave a tour of the parts of the system that are, in her experience, particularly likely to trip things up. At the top of the list was graphics processors (GPUs); these are complex devices and it tends to take quite a while to get power management working properly on them. The "RC6" mechanism describes a set of power states for GPUs; naturally, one wants the GPU to be in a low-power state when it doesn't have much to do. Beyond that, framebuffer compression can reduce memory bandwidth use depending on what's in the framebuffer; sending less video data results in lower power usage. Kristen suggested that users choose a simple (highly compressible) background image on their devices for the best power behavior. "Panel self refresh" allows the CPU to stop sending video data to the screen entirely if things are not changing; it can be inhibited by things like animated images on the screen. 

Another "problem child" is audio. On many systems, audio data can be routed through the GPU, preventing it from going into an idle state. Audio devices tend to be complex, consisting of, at a minimum, a controller and a codec; drivers must manage power-management states for both of those devices together. 

On the USB side, the USB 3.0 specification added a number of useful power-management features. USB 2.0 had a "selective suspend" feature, but it adds a lot of latency, reducing its usefulness. In 3.0, the controller can suspend itself, but only if all connected devices are suspended. The USB "link power management" mechanism can detect low levels of activity and reduce power usage. 

There are three power-management technologies available for SATA devices. The link power management mechanism can put devices into a sleep state and, if warranted, turn the bus off entirely. "ZPODD" is power management for optical devices, but Kristen has never seen anybody actually use it; optical devices are, in general, not as prevalent as they once were. The SATA controller itself offers some power-management features, but they tend to be problematic, she said, so they are not much used in Linux. 

The PCI Express bus has a number of power-management options, including ASPM for link-level management, RTPM as a runtime power-management feature, and latency tolerance reporting. The I2C bus has fewer features, being a simpler bus, but it is usually possible to power-down I2C controllers. Human-input devices, which are often connected via I2C, tend to stay powered up while they are open, which can be a problem for system-wide power management. 

And, of course, software activity can keep a system from going into deep idle states. If processes insist on running, the CPU will stay active, leaving suspend as the only viable option for power savings. Even brief periods running in the CPU can, if they cause it to wake from idle often, significantly reduce battery life. 

#### Idle together

The conclusion from all of this is that power management requires a coordinated effort. For a system to go into a low-power state, a number of things must happen. User space must be quiet, the platform must support low-power states across all devices, and the kernel must properly support each device's power-management features. The system must also be configured properly; Kristen expressed frustration at mainstream distributions that fail to set up proper power management at installation time, wasting the effort that has been put into power-management support at the lower levels. Getting all of the pieces to work together properly can be a difficult task, but the result — systems that efficiently run our most important workload — is worth the trouble. 

[Your editor would like to thank the Linux Foundation for funding his travel to LinuxCon Japan]
