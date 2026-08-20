---
title: Putting a lid on USB power
url: https://lwn.net/Articles/186331/
date: "June 5, 2006"
category: "Development model-Kernel quality; USB"
author: 
---

Kernel bugs are bad news. Among the worst bugs are regressions - situations where a once-working system breaks after a kernel upgrade. The kernel developers have been taking an increasingly hard line against regressions; patches which break working systems will usually be reverted, even if those patches fix other problems. The idea, as pushed by Linus, is that once a system works, it should continue to work into the future. 

As it happens, a number of USB users have found that, on upgrading to 2.6.16, their systems do not work anymore. But, in this case, this "regression" is not seen as such by the developers and is not likely to change. This issue is a good demonstration of the sort of tradeoffs which operating systems developers must make. 

USB ports can supply power to the devices plugged into them; this power is sufficient to drive many devices, as well as totally unrelated items (such as USB-powered LED lamps). There are limits to the amount of power which can be supplied, however. USB devices will communicate their maximum current draw to the host, which can then decide whether it has the capacity available or not. If sufficient power is not available, the device will not be allowed to configure itself and operate. 

There are many rules in the USB specification on how power configuration should work. One of those applies to unpowered USB hubs - the ones which lack a power supply of their own. The total current drawn by an unpowered hub cannot be allowed to exceed what the host can supply; in particular, the USB specification limits devices on unpowered USB hubs to 100 mA of current. Even if only one hub port is in use, that single port is limited to that value, despite the fact that a larger draw should work in that situation. 

Prior to 2.6.16, the Linux kernel did not actually check power requirements before configuring devices. With 2.6.16, however, any device whose stated maximum power requirement exceeds 100 mA will not be allowed to configure itself on an unpowered hub. Thus, devices which worked in that mode in earlier kernels now fail to operate; not all users are entirely pleased. 

The argument has been made that, since these configurations almost always work in the real world, the kernel should not be shutting them down now. The fact is, however, that running hardware outside of its specifications is always a dangerous thing to do. Often one will get away with it, but sometimes things can fail badly. A fairly large class of USB devices are mass storage devices; the consequences of power-related problems with these devices could include corrupted data and damaged hardware. These are not consequences which the USB developers wish to inflict on their users, so, instead, they refuse to operate devices out of their specifications. 

To the developers, the fact that some previously-working hardware now fails to operate is not a regression. It is a bug fix, with the kernel finally performing some due diligence which should have been happening all along. They do not intend to change this behavior. 

As it happens, it _is_ possible to convince the kernel to override its good sense and configure the device anyway. It is not easy, however. Essentially, the steps are this: 

  * Run `lsusb -v` and find the entry for the device of interest. Your editor's USB mouse, for example, is described by an entry starting "`Bus 001 Device 003: ID 046d:c01b Logitech, Inc. MX310 Optical Mouse`". This mouse is plugged into a hub listed previously as being "Bus 001, Device 002". Together, these numbers turn into a path number "`1-2.3`". This number is important. 

  * Under that same device entry will be found one or more possible device configurations, along with their associated power requirements. Each of these configurations includes a `bConfigurationValue` number describing it. The number associated with the desired configuration must be found; in many cases it is one. 

  * Force the device configuration with a line like: 

```
echo -n 1 > /sys/bus/usb/devices/1-2.3/bConfigurationValue
```

The configuration values and path number must be replaced with the actual values determined from the `lsusb` output. 

Needless to say, this sequence of steps is not entirely easy - and it must be repeated each time the device is plugged in. For those who are comfortable writing `udev` rules, this configuration change can be automated without too much trouble. Perhaps the desktop environments will eventually be made smart enough to detect this situation and offer (with suitable scary warnings) to override the kernel for specific devices. But it might just be better to buy a powered hub or plug the device directly into the host.
