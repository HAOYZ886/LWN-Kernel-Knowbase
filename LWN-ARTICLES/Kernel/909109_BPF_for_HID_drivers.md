---
title: BPF for HID drivers
url: https://lwn.net/Articles/909109/
date: "September 26, 2022"
category: "BPF-Device drivers"
author: "By Jonathan Corbet September 26, 2022 LPC"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
September 26, 2022

* * *

[LPC](<https://lwn.net/Archives/ConferenceByYear/#2022-Linux_Plumbers_Conference>)

The [Human Interface Device](<https://en.wikipedia.org/wiki/Human_interface_device>) (HID) standard dates back to the Windows 95 era. It describes how devices like mice and keyboards present themselves to the host computer, and has created a world where a single driver can handle a wide variety of devices from multiple manufacturers. Or it would have, if there weren't actual device manufacturers involved. In the real world, devices stretch and break the standard, each in its own special way. At the [2022 Linux Plumbers Conference](<https://lpc.events>), Benjamin Tissoires described how BPF can be used to simplify the task of supporting HID devices. 

Most devices, he began, will work just fine with the kernel's generic HID drivers. That still leaves quite a few that present problems — behavioral quirks that require a special driver to address. Most of the time, that driver need only make a few tweaks to the "report descriptor" provided by the device. This descriptor, the format of which was defined in 2001, describes the exact protocol a device speaks and which capabilities it offers. The kernel contains a long list of tiny drivers that do little beyond tweaking a device's report descriptor to make it adhere to the standard; see [`drivers/hid/hid-sigmamicro.c`](<https://elixir.bootlin.com/linux/v5.19.10/source/drivers/hid/hid-sigmamicro.c>) for an example. Others, only slightly more complex, will modify input events upon receipt from the device; [`drivers/hid/hid-ezkey.c`](<https://elixir.bootlin.com/linux/v5.19.10/source/drivers/hid/hid-ezkey.c>) shows that type of manipulation. 

[![\[Benjamin
Tissoires\]](https://static.lwn.net/images/conf/2022/lpc/BenjaminTissoires-sm.png)](<https://lwn.net/Articles/909115/>) Device manufacturers, of course, show no sign of running out of ideas for new ways to make broken hardware, so the kernel will continue to need to fix things up in new ways. Currently, each quirk fix requires the writing of a new driver, which must then go through the usual kernel review process before getting upstream and, some time later, onto the systems where the offending device is actually used. This is not a great experience for users and creates work for developers; it gets worse if developers lack access to the device in question and must rely on users to build kernels to test proposed fixes. If there were a way to just describe the tweaks needed for a given device, then new devices could be supported quickly on existing kernels, without adding more kernel modules. 

That way, of course, is BPF. The idea behind [Tissoires's work](<https://lwn.net/ml/linux-kernel/20220824134055.1328882-1-benjamin.tissoires@redhat.com/>), which seems likely to be merged for 6.1, is to make it possible to easily create a small program to make a new device work. That program could be dropped into a directory, from which it would be loaded into the kernel. Users will not need to worry about building kernels, and developers can avoid adding more little modules. Instead, the plan is to add these BPF programs to the upstream kernel as needed to support new devices. 

But, Tissoires said, once it becomes possible to modify device behavior with BPF, there are other interesting things that can be done. One of those was described as the "HID firewall". Steam, he said, makes game controllers accessible to any process running on the system; a malicious program could rewrite a device's firmware in ways that are unlikely to improve the owner's player ranking. A simple BPF program could block access to the firmware-update endpoint on the device, preventing such attacks. 

It is also possible to transform devices into something different. The [Microsoft Surface Dial](<https://support.microsoft.com/en-us/surface/meet-surface-dial-1e58a0e6-4d4a-6303-afcd-ef0234047628>), he said, is an interesting input device but, because it is new and different, no software supports it. A BPF program could tweak data coming from the device to make it appear to be a mouse instead, making it usable with existing software. 

BPF can also be helpful in debugging problems with HID devices. The `hidraw` device provides useful data now, he said, but it only shows data from a device; there is no way to see the accesses _to_ that device. BPF would make it easy to trace the full interaction with a HID device. 

BPF programs are executed on data from the device before it is processed by the HID core, he said; that makes it possible for them to modify that data. Multiple programs can be attached to a single device, but the order in which they will be executed is "undefined". Beyond tweaking data, BPF programs can do things like filtering out spurious button clicks. It will also be possible for BPF programs to communicate directly with devices. 

The implementation relies on tracepoints and, specifically, the error-injection capability. Tracepoints are added at places where changes can usefully be made: reading the report descriptor, receipt of input events, and on a user system call. A set of kfuncs has been provided to facilitate communication with the device; [this documentation patch](<https://lwn.net/ml/linux-kernel/20220824134055.1328882-26-benjamin.tissoires@redhat.com/>) describes the BPF interface in detail. 

An important limitation, Tissoires said, is that BPF cannot be used to fix devices that are broken (and needed) at boot time. Specifically, that rules out using BPF for most fixes applying to keyboards. 

One open question, he said in conclusion, is determining the best method for shipping device fixes with the kernel. One approach would be to create a separate module for each BPF source file, but that, once again, leads to the creation of a lot of modules. ("Module" was his word; he may have been speaking about loader programs that run in user space, though). Alternatives would be to create one big module with a lot of BPF programs, or to just ship the fixup programs from the firmware repository instead. 

[Thanks to LWN readers for supporting my travel to this event.]
