---
title: "LinuxCon: x86 platform drivers"
url: https://lwn.net/Articles/456146/
date: "August 24, 2011"
category: "Device drivers-Platform drivers"
author: "By Jake Edge August 24, 2011"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
August 24, 2011

With his characteristically dry British humor, Matthew Garrett outlined the current situation with x86 platform drivers at [LinuxCon](<http://events.linuxfoundation.org/events/linuxcon>). These drivers are needed to handle various "extra" hardware devices, like special keys, backlight control, extended battery information, fans, and so on. There are a wide range of control mechanisms that hardware vendors use for these devices, and, even when the controller hardware is the same, different vendors will choose different mechanisms to talk to the devices. It is a complicated situation that seems to require humor--and perhaps alcohol--to master. 

[ ![\[Matthew Garrett\]](https://static.lwn.net/images/2011/lcna-garrett-sm.jpg) ](<https://lwn.net/Articles/456218/>)

Garrett does a variety of things for Red Hat, including hardware support and firmware interfaces (e.g. [for EFI](<https://lwn.net/Articles/454399/>)). Mostly he does ""stuff that nobody else is really enthusiastic about doing"", he said. Platform drivers are ""bits of hardware support code"" that are required to make all of the different pieces of modern hardware function with Linux. Today's hardware is not the PC of old and it requires code to make things work, especially for mobile devices. 

He started by looking at keys, those used to type with, but also those that alter display brightness or turn hardware (e.g. wireless) on and off. The "normal" way that keys have been handled is that a key press causes an interrupt, the kernel reads a value from the keyboard controller, and the keycode gets sent on to user space. The same thing happens for a key up event. This is cutting edge technology from ""1843 or something"", which is very difficult to get wrong, though some manufacturers still manage to do so. The first thing anyone writes when creating a ""toy OS"" is the keyboard driver because it is so straightforward. 

In contrast to that simple picture, Garrett then described what goes on for getting key event information on a Sony laptop. The description was rather baroque and spanned three separate slides. Essentially, the key causes an ACPI interrupt, which requires the kernel to do a multi-step process executing "general purpose event" (GPE) code in the ACPI firmware, and calling ACPI methods to eventually get a key code that ends up being sent to user space. ""This is called value add"", he said. 

Manufacturers are convinced that you don't want to manage WiFi the same way on multiple devices. Instead, they believe you want to use the "Lenovo wireless manager" (for example) to configure the wireless device. ""Some would call them insane"", and Garrett is definitely in that camp. The motivation seems to be an opportunity for the device maker to splash their logo onto the screen when the manager program is run. As might be guessed, there is no documentation available because that would allow others to copy the implementation, which obviates the supposed value add. 

It is not just keyboards that require platform drivers, Garrett said. Controlling radios, ambient light sensors (""everyone wants the brightness to change when someone walks behind them""), extended battery information (using identical battery controller chips, with the interface implemented differently on each one), hard drive protection (which always use the same accelerometer device), backlight control, CPU temperature, fan control, LEDs (e.g. a "you have mail" indicator, that is ""not really useful"" but is exposed ""for people who don't have anything better to do with their lives""), and more, all need these drivers. 

#### Multiple control mechanisms

There are half-a-dozen different interfaces that these drivers will use to control the hardware, starting with plain ACPI calls. That is generally one of the easiest methods to use, because it is relatively straightforward to read the ACPI tables and generate a driver from that information. Events are sent to the driver, along with an event type, and some reverse engineering is required to work out what the types are and what they do. There are specific ACPI calls to get more information about the event as well. Garrett's example showed two `acpi_evaluate_object()` calls for the AUSB ("attach USB") and BTPO ("Bluetooth power on") ACPI methods, which is all that is needed to turn on Bluetooth for a Toshiba device. ""Wonderful"", he said. 

A small micro-controller with closed-source firmware--the embedded controller--is another means to control hardware. Ideally, you shouldn't have to touch the embedded controller because ACPI methods are often provided to do so. But, sometimes you need to access the registers of the controller to fiddle with GPIO lines or read sensor data stored there. The problem is that these register locations can and do change between BIOS versions. While it is ""considered bad form to write a driver for a specific BIOS version"", sometimes you have to do so. It is a fairly fragile interface, he said. 

Windows Management Instrumentation (WMI) is a part of the Windows driver model that Microsoft decided would be nice to glue into ACPI. It has methods that are based on globally unique IDs (GUIDs) corresponding to events. A notify handler is registered for a GUID and it gets called when that event happens. The Managed Object Format (MOF) code that comes with a given WMI implementation is supposed to be self-documenting, but there is a problem: it is compressed inside the BIOS using a Microsoft proprietary compression tool ""that we don't know how to decompress"". As an example of WMI-based driver, Garrett showed a Dell laptop keyboard handling driver that reports the exact same keycode that would have come from a normal keyboard controller, but was routed through WMI instead, ""because this is the future"", he said. 

Drivers might also be required to make direct BIOS calls, which necessitates the use of a real mode `int` instruction. This is ""amazingly fragile"" and incompatible with 64-bit processors. Currently, the only time BIOS interrupts are invoked from user space are for X servers and Garrett suggests that drivers should ""never do this"". In fact, he went further than that: ""If you ever find hardware that does this, tell me and I will send you money for new hardware"". If you decide to write code that implements this instead, he said that he would pay someone else money to ""set fire to your house"". 

System Management Mode (SMM) traps are yet another way to control hardware, but there seems to be a lot of magic involved. There are ""magic addresses"" that refer to memory that is hidden from the kernel. In order to use them, a buffer is set up and the address is poked, at which point the ""buffer contents magically change"". There have been various problems with the SMM implementations from hardware vendors including some HP hardware that would get confused if SMM was invoked from anything other than CPU 0. Garrett did not seem particularly enamored of this technique, likening it to the business plan of the "[Underpants Gnomes](<http://en.wikipedia.org/wiki/Gnomes_%28South_Park%29>)". 

The last control mechanism Garrett mentioned is to use a native driver to access the hardware resources directly. Typically these drivers use ACPI to identify that the hardware exists. The hardware is accessed using the port IO calls (i.e. `inb()`, `outb()`), and will use native interrupts to signal events. Various models of Apple hardware uses these kinds of drivers, Garrett said. 

#### Consistent interfaces

While there are many ways to access the hardware, kernel hackers want to provide a consistent interface to these devices. We don't want ""to have to install the Sony program to deal with WiFi"". So, "hotkeys" are sent through the input system, ""keys are keys"". Backlight control is done via the backlight class. Radio control is handled with rfkill, thermal and fan state via hwmon, and the LED control using the led class. That way, users are insulated from the underlying details of how their particular hardware implements these functions. 

There are two areas that still have inconsistent interfaces, Garrett said. The hard drive protection feature that is meant to park the disk heads when an untoward acceleration is detected (e.g. the laptop is dropped) does not have a consistent kernel interface. Also, the [ambient light sensors are lacking an interface](<https://lwn.net/Articles/390634/>). The latter has become something of a running joke in the kernel community, he said, because Linus Torvalds thinks it should be done one way, but the maintainer disagrees, so, as yet, there is no consistent interface. 

#### How do I work this?

Garrett also had some suggestions on figuring out how new/unsupported hardware is wired up. There is a fair amount of reverse engineering that must be done, but the starting point is to use `acpidump` and `acpixtract` utilities to find out what is in the ACPI code in the hardware. 

If the device is WMI-based, `wmidump` may also be useful. Extracting the event GUIDs and registering a handler for each will allow one to observe which ones fire for various external events. Then it is a matter of flipping switches to see what happens, parsing the data that is provided with the event, and figuring how to do something useful. This may require alcohol, he said. 

For embedded controllers or direct hardware access, there are sysfs files that can be useful. The embedded controller can be accessed via `/sys/kernel/debug/ec/ec0/io` (at least for those who have debugfs mounted), or by using the `ec_access` utility. Once again, you need to hit buttons, throw various switches, and listen for fan changes. In addition, you should test that the register offsets are stable for various machine and BIOS version combinations, he said. You can find the IDs of devices to access them directly via the `/sys/bus/pnp/devices/*/id` files, register as a PNP bus driver for devices of interest, and then ""work out how to drive the hardware"". 

The overall picture that Garrett painted is one of needless complexity and inconsistency that is promulgated by the hardware vendors. But, it is something that needs to be handled so that all of the "extras" baked into today's hardware work reliably--consistently--with Linux. While it would be nice if all of these components were documented in ways that Linux driver writers could use, that doesn't seem likely to change anytime soon. Until then, Garrett and the rest of the kernel community will be wrestling with these devices so that we don't get locked into manufacturer-specific control programs. 

[ I would like to thank the Linux Foundation for travel assistance to attend LinuxCon. ]
