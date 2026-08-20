---
title: "USB charging, part 2: implementation"
url: https://lwn.net/Articles/694062/
date: "July 13, 2016"
category: USB
author: "July 13, 2016 This article was contributed by Neil Brown"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

July 13, 2016

This article was contributed by Neil Brown

In the [first part of this series](<https://lwn.net/Articles/693027/>) we explored the complexities of charging a battery in a portable Linux-driven device from a USB connection, and in particular looked at how the maximum allowed current can be determined. This resulted in five tasks that Linux would need to complete in order to charge batteries in a compliant manner. It is now time to look inside Linux to see how well it achieves these tasks and, as we will find, the answer is "not very well", or at least "not very uniformly". There is some reason for hope on the horizon, however, as a patch set described as providing a "usb charger framework" is under development and should close at least some of the gaps.

The five tasks we identified, and that we will address in order, are:

  * find out from the USB PHY what type of cable is attached and report this to the battery charger
  * advertise USB gadget configurations with appropriate power demands
  * determine which gadget configuration was chosen and report the available power to the battery charger
  * adjust current within the given range to maintain suitable voltage
  * detect when the power supply is questionable during boot, and limit activation of components until that is resolved

#### The EXTernal CONnector in your USB PHYsical interface

When a cable is plugged into the B-series USB receptacle on your device, it is the task for the PHY, and the Linux driver for the PHY, to measure voltage levels and resistances to determine what sort of cable has been plugged in. The PHY driver must then tell the USB core code if it should start negotiations as a USB host or a USB gadget; it must also report the cable type to whatever driver is responsible for charging the battery. How these reports are sent could best be described as _ad hoc_ , though a less kind commentator might say it is a total mess. There are two approaches that are fairly generic: one legacy and one newer. And then there are non-generic approaches like [`musb_mailbox()`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/drivers/usb/musb/musb_core.c?id=a99cde438de0c4c0cecc1d1af1a55a75b10bfdef#n1683>).

The legacy approach requires that the charger call `usb_register_notifier()`, as eight charger drivers do. The [notifier](<https://lwn.net/Articles/160953/>) mechanism allows a pointer to an arbitrary data structure to be passed along with the notification. Some PHY drivers pass a pointer to an integer giving the available current in mA, some pass a pointer to the `usb_gadget` structure, which doesn't contain any information about available current, and some just pass `NULL`. Even without any data passed, the notification can be useful since the charger driver may be able to query the PHY directly, and can almost certainly turn the charging circuit on or off depending on whether there is any voltage. So, while this is not a coherent interface, it does provide some value.

The newer approach is to use "extcon", which is a driver class for monitoring external connectors, whether for audio jacks, video ports, USB receptacles, or anything else. An extcon device maintains a record of what type of cable (or what collection of cables) is currently plugged in and will generate a notification whenever a cable is plugged or unplugged. Other drivers can register interest in a particular cable type being attached to a particular connector or in a particular cable type being attached to any connector. Strangely, there is no way to register interest in a particular connector regardless of cable type.

Among [the cable types known to extcon](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/include/linux/extcon.h?id=a99cde438de0c4c0cecc1d1af1a55a75b10bfdef#n30>) are:

```
/* USB external connector */
            #define EXTCON_USB		1
            #define EXTCON_USB_HOST		2
    
            /* Charging external connector */
            #define EXTCON_CHG_USB_SDP	5	/* Standard Downstream Port */
            #define EXTCON_CHG_USB_DCP	6	/* Dedicated Charging Port */
            #define EXTCON_CHG_USB_CDP	7	/* Charging Downstream Port */
            #define EXTCON_CHG_USB_ACA	8	/* Accessory Charger Adapter */
            #define EXTCON_CHG_USB_FAST	9
            #define EXTCON_CHG_USB_SLOW	10
```

Unfortunately, there is no documentation beyond what is given above and the implicit documentation of how various drivers use the cable types. `EXTCON_CHG_USB_SLOW` seems to suggest a cable that can provide 500mA. `EXTCON_CHG_USB_FAST` is used by `axp288_charger.c` to indicate a charger capable of 2000mA. The relationship between the `EXTCON_USB*` and `EXTCON_CHG_USB_*` cable types seems confused.

A possible interpretation is that the `EXTCON_USB*` cable types indicate if a cable can carry data, either in gadget or host mode, independent of any charging capabilities. The `EXTCON_CHG_USB_*` types would then indicate the power that can be expected of the cable, independent of any data. Thus a single USB cable might be reported as both a data cable and a power cable, which certainly makes it easier for any client that is only interested in one or the other. A few drivers, such as `extcon-max14577.c`, report a standard downstream port as both `EXTCON_USB` and `EXTCON_CHG_USB_SDP`, which supports this hypothesis, but, since they don't report `EXTCON_USB` together with `EXTCON_CHG_USB_CDP` or `EXTCON_USB_HOST` together with `EXTCON_CHG_USB_ACA`, this is not an interpretation that can safely be relied upon.

Even though these cable definitions do not seem to be implemented consistently, there is infrastructure available that carries all the information we need. Updating some drivers to use existing infrastructure properly is a trivial task compared to trying to work out what infrastructure is needed to allow the drivers to communicate at all.

And, indeed, drivers would need to be updated. There are precisely two charger drivers that listen for extcon notifications. Quite a few USB drivers listen for `EXTCON_USB` or `EXTCON_USB_HOST` so they can configure as a gadget or a host, but the only chargers that do are `axp288_charger.c` and `charger_manager.c`.

It is from `axp288_charger.c` that we can discover the one interpretation of `EXTCON_CHG_USB_FAST` and `EXTCON_CHG_USB_SLOW` that was mentioned above, but otherwise it isn't particularly helpful as the code doesn't appear to work. The API for extcon was updated last year and when `axp288_charger.c` [was adjusted to match](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=4b703b1d4c46ca4a00109ca1a391943ec21991b3>), the only improvement provided was the removal of compiler errors.

`charger_manager.c` is a software battery-charge monitor that checks the temperature and voltage on a battery and decides when to try to charge it. It can be configured to expect a list of different cable types along with the current to try to use from each cable. This seems to be the closest thing to a working charger manager that uses an extcon device to be notified of cables.

This poor state of the code doesn't necessarily mean that no Linux device charges properly over USB. The USB PHY and the charging controller in a particular device are often from the same manufacturer and even in the same integrated circuit. In these cases, a driver for one half can have intimate knowledge of the other half and thus achieve reasonable results. An example of such a driver is `isp1704_charger.c`. This driver is ostensibly a driver for battery charging, but it reaches over into the territory of the PHY driver to [directly access "ULPI" registers](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/drivers/power/isp1704_charger.c?id=a99cde438de0c4c0cecc1d1af1a55a75b10bfdef#n97>), which is the USB Low Pin Interface. It uses `usb_register_notifier()` to find out when something changes, then pokes around on its own to see the specifics of the change.

Where I have mentioned "charger drivers" above I have been a little loose with terminology. Linux doesn't have a "battery charger" class for drivers, it only has a "`power_supply`" class. The unifying feature of this class is that it allows drivers to report various details of a power source, such as voltage (both present and maximum), current, capacity (for batteries), technology used, etc. Since the most important aspect of charging a battery is managing the source of power, and possibly turning it off when temperature or voltage monitors indicate a problem, it is quite reasonable for battery charging to be managed by power supply devices, and this is how it happens in Linux.

One of the properties a power supply can present is the supply type, and until 2010 it was one of battery, UPS, mains, or USB. At that time [USB DCP, USB CDP and USB ACA were added](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=85efc8a18cedf70e55acd0c825e2d9d2f3b19999>). More recently, [some more types specific to USB 3.0 have been added](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=6659b55dae09b2042d8cdd0f4576502c3e81957c>). This means we have two subsystems vying for ownership of the USB-charger-type concept. Is the type of charger plugged into a USB receptacle a property of the power supply, or a property of the cable (or external connection)? Or both? The "technology" property mentioned previously is currently used only for batteries, allowing NiMH, LION, NiCd, etc. If the power supply needs to know about the attached charger, rather than just being told the available current, should the various types be treated in the same way as battery technology? While it is doubtless possible to argue for various different options, it is hard to argue against having unified coherent usage and that is certainly missing.

The various USB power supply subtypes are currently used in three different drivers. The `axp288_charger.c` that we have already met uses some of the values, but just uses them internally. It doesn't use them to report the type of the power supply (that is always `POWER_SUPPLY_TYPE_USB`) but [stores them in a data structure called `cable`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/drivers/power/axp288_charger.c?id=a99cde438de0c4c0cecc1d1af1a55a75b10bfdef#n165>). It finds out the type of charger by registering with an extcon device, but as already noted that doesn't work correctly. So that driver isn't good example to learn from.

Then there is a `gpio-charger.c`, which is designed to work with power-supply hardware with limited monitoring options: a GPIO input can detect if the charger is active, but that is all. In order to provide the other properties that a power supply should have, `gpio-charger.c` reads some configuration information from a device-tree description of the hardware. It allows that description to declare that the power supply is some particular subtype of USB. But this type is not changed dynamically, so it could only be meaningful for a USB charger that was hardwired to the device, which seems a little pointless.

Finally there is the `isp1704_charger.c`. As mentioned, it is a power-supply driver that pokes in the USB registers to determine the power-supply type, which is a bit of a layering violation. So it seems that no power-supply driver in mainline actually uses the USB power-supply subtypes in a particularly useful way.

So let's move on to determining current usage during bus enumeration.

#### Tracking gadget configuration

When a Standard Downstream Port (SDP) connection is detected, the PHY driver notifies the USB gadget controller, which then proceeds with the enumeration process. The parts of this that interest us are how `MaxPower` values are chosen and how the `MaxPower` from the chosen configuration is communicated. `MaxPower` is the field in a USB configuration table that lists the current requirement, which can be seen using: 

```
lsusb -v | grep MaxPower
```

Linux provides a "composite" gadget design where a number of different drivers can each register their own configuration and the composite driver will provide a list of all of those configurations to the host for it to choose from. There is a serial driver, an ether driver for networking, a mass_storage driver, and several others. Each of these just provides a single configuration and, while a few do set the `MaxPower` field in that configuration, most just leave it as the default. This default can be set using the compile-time configuration option `CONFIG_USB_GADGET_VBUS_DRAW`. This option defaults to 2mA, which is the smallest non-zero number that can be represented; zero is technically legal but apparently confuses some hosts. `CONFIG_USB_GADGET_VBUS_DRAW` is the sort of number that doesn't really make sense as a configuration option, but was probably implemented that way because it was easier than finding a real solution. No attempt is made to offer multiple versions of each configuration with different power requirements as was suggested in the previous article.

It may be possible to offer multiple configurations by a different route. The composite USB gadget can be configured at runtime using `configfs`. As [these slides [PDF]](<https://events.linuxfoundation.org/sites/events/files/slides/LinuxConNA2013-andrzej.pietrasiewicz-usb-gadget-configfs_0.pdf>) describe, it is possible to create multiple configurations and set the `MaxPower` for each. This interface could be used to create multiple configurations for each driver, but that does feel a little roundabout and clumsy.

Whatever configuration is created, once it has been chosen by the host, the core USB gadget driver will report this information to the hardware-specific gadget driver by calling the `vbus_draw()` method on that driver. Of those gadget drivers that actually provide a `vbus_draw()` method (some don't) and don't simply ignore the value (several do), most just call `usb_phy_set_power()` to tell the PHY driver what power is available. If that sounds like passing the buck to you, I would agree. Most PHY drivers just ignore the number too.

One exception is the [`s3c2410_udc.c` USB gadget driver](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/drivers/usb/gadget/udc/s3c2410_udc.c?id=4c2e07c6a29e0129e975727b9f57eede813eea85#n1508>) used in the GTA02, which is the original OpenMoko phone. It calls a function provided by the "board" file that contains specifics of the particular platform. The GTA02 board file [uses a private mechanism to pass the number to the power manager](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/arch/arm/mach-s3c24xx/mach-gta02.c?id=4c2e07c6a29e0129e975727b9f57eede813eea85#n221>). It is probable that out-of-tree drivers in vendor kernels use a similar approach.

#### Setting the right current

Once the current that might be available has been determined and communicated to the charging manager, it is necessary to configure the charging power supply with an appropriate current, preferably the highest permitted current that doesn't cause the voltage to drop too low. As far as I can tell from exploring the code, there is only one driver that tries anything more sophisticated than setting a fixed current level, possibly dependent on the type of cable or `vbus_draw()` setting. That driver is the `twl4030_charger.c` that drives the battery charger in the OpenPhoenux GTA04; I know about that driver, and its imperfections, because I wrote the code to control the current.

The code in this driver increases the current requested in steps of 20mA until the voltage drops to 4.75V or until the maximum permitted is reached. This process mostly works, but subsequent reflections revealed a problem. If the battery is fully charged, then the phone as a whole cannot make use of more that a few hundred mA, so increasing the current setting won't actually put more load on the power supply, and thus won't cause the voltage to drop. This could lead to the current request being set to the maximum permitted even if it exceeds the maximum available. The charging hardware stops feeding current to the battery when the battery voltage reaches a certain level and the battery will be allowed to power at least some of the hardware. After the voltage drops a little, the charging turns back on, and at this point the battery may be able to accept more current than it could when the available current was being measured. This current might overload the charger.

The main point about this code is that it is easy to get wrong, but in principle should be common to all chargers that can limit current and measure voltage. So it really belongs in a common location — but where? There do seem to be a number of different elements of functionality needed for USB charging and they are currently implemented in an _ad hoc_ manner. Bringing it all under a common umbrella appears to be the goal of USB charger framework that is currently being developed by Baolin Wang; it was [recently posted in its 15th revision](<https://lkml.org/lkml/2016/7/1/35>).

#### The USB charger framework

The framework attaches a "usb charger" object to every USB gadget device that is created and intercepts the `vbus_draw()` calls so that it knows when an SDP has been configured. If the USB gadget device is described in the device-tree as having an "extcon" connector attached, it will register for notifications for cable-change events.

Other drivers, such as a charger driver, can register to receive notifications from a USB charger if they know the name of the charger. The name will always be `usb-charger.0` unless there are multiple chargers. When any change happens to the charger, it will notify all registered listeners to tell them the new current limit. This limit is a single number, not a range, so it needs to be handled carefully. 

If charger managers were required to increase the current gradually up to this level, then sending the maximum would be appropriate. If they were expected to always enable exactly this number, then sending the minimum is the only safe approach. In the default configuration, the framework advises a current limit of 1500mA for the various types of chargers. This is the maximum for some, but not all, cable types. The only example of a charger driver that has been modified to use this information simply sets the limit rather than carefully ramping up to the limit. This may be safe, but only if that hardware has its own built-in current ramping.

When the framework registers interest in an extcon, it only requests notification of `EXTCON_USB` cables, not the various charger cables. When that notification arrives, it checks with the configured power supply to see what USB subtype it is and reports available current based on that. So this framework seems to have sided with USB cable types being the property of the power supply rather than the property of the cable.

#### Conclusion

While most of the parts needed for compliant USB charging are present, they are not implemented consistently, and it isn't even entirely clear what the right approach should be even if the USB charging framework does get merged. That wasn't the answer I was hoping for when I started examining this issue, but does at least clarify the current situation. Knowing where we stand makes moving forward a lot easier.

The one question I haven't yet covered is the need to keep most devices off until a stable source of power is assured. The reason for keeping this separate is that it is unlikely to ever be a part of Linux. There are enough interdependencies between discovery of different devices in Linux that trying to delay some in unusual circumstances is likely to lead to hard-to-diagnose problems. 

Since the device is encouraged to avoid any unnecessary tasks until power is stable, it makes sense to not even boot Linux straight away. U-Boot, a common boot loader for mobile devices, is sufficiently powerful to be able to handle all the necessary negotiation to enable the maximum current possible. It should then be able to enter a low-power state until the battery has reached a sufficient charge to carry all the way through the Linux boot process. Linux will likely turn off battery charging during boot and renegotiate from scratch, so the battery needs enough charge to get all the way through the boot on its own. 

There is clearly plenty to do to get USB charging into a healthy state. A particularly valuable first step would be to get clarity on how extcon and `power_supply` devices should work with the different cable types, and then to provide a standard way for charging power supply devices to ramp up the load while maintaining adequate voltage. With these in place, individual drivers could be updated to use these newly clarified interfaces on an as-needed basis. It's just a small matter of programming.
