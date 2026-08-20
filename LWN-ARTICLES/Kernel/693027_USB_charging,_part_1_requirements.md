---
title: "USB charging, part 1: requirements"
url: https://lwn.net/Articles/693027/
date: "July 7, 2016"
category: USB
author: "July 7, 2016 This article was contributed by Neil Brown"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

July 7, 2016

This article was contributed by Neil Brown

USB, the Universal Serial Bus, was primarily designed to transfer data to and from peripherals, with a secondary function of providing power to those peripherals so that they don't need to be independently powered. This secondary function has gained importance over the years, so that today it is sometimes the primary or only function. Many smartphone owners regularly use USB to charge their device but rarely, if ever, transfer data over USB. When Linux is running on that smartphone it needs to be able to optimize the use of whatever power is available over the bus — a task that is neither straightforward nor cleanly implemented in mainline Linux. We start this two-part series by looking at how USB communicates power availability information and will conclude in the second part by looking at how Linux does, or more often doesn't, make use of this information.

To begin, it will be helpful to be clear about some terminology. USB is an asymmetric bus — the two connected peers each play different roles. One peer is the master, or initiator, and controls all the data flow; it is known as the "host" and has an A-series connector (or receptacle) or, possibly, a B-series plug. The other peer is the slave, or responder, that can only send or receive data when the host tells it to. This is known as a USB "device" though, since "device" is an overly generic term, it is sometimes referred to as a "gadget". A USB gadget has a B-series connector, of which there are a range of sizes, or an A-series plug.

A USB cable connecting a host to a gadget typically has 4 wires. Two carry power, VBUS and GND, and two carry data, DP and DM, also known as D+ and D-. Power normally flows from the host to the gadget, but this is not universal as we shall see later. USB 3.0 adds extra wires and pins to the A-series and B-series connectors to carry "SuperSpeed" data, but only changes power delivery in that power flowing in different directions can flow over different wires. The USB 3.0 C-series cable is bidirectional and does add extra power signaling options, but the details of that would not particularly help the present discussion. 

Some USB peers can serve as either a host or a gadget using a specification known as USB On-The-Go — USB-OTG. These devices have a B-series connector with a fifth pin called ID. The port will act as a gadget port unless a cable is plugged that connects the ID pin to GND, possibly through a resistor. When that happens, the device will switch the port to host mode so it can control an attached gadget.

From the perspective of a mobile battery-powered Linux device with a B-series port, the important question is: how much power can be drained from the bus and used in the device? As the voltage is fixed at 5V ±5% this is equivalent to a question of how much current can be drained, and the answer is usually given in milliamps (mA). The device will typically have power-management circuitry that can limit the current used, and other circuitry that will divert some to charging the battery when that is appropriate, but those details are not really important at this stage. For now, we only care about a number.

The [USB Implementers Forum](<http://www.usb.org/>) provides several multi-page specifications describing how to get that number, in particular the Battery Charging v1.2 document ("BC-1.2", which is my primary source and comes from the [Class Specification page](<http://www.usb.org/developers/docs/devclass_docs/>)) and the newer [USB Power Delivery](<http://www.usb.org/developers/powerdelivery/>) spec. In practice, there are two classes of answers.

#### Current from a Standard Downstream Port

The first class of answers applies when the device is connected by a standard cable to a standard A-series host port such as on a notebook or desktop computer. A USB host provides a "Standard Downstream Port" (SDP). These ports must provide 100mA for at least the first second after attachment and can be configured to deliver more — up to 500mA in USB-2 — after enumeration has completed. If no enumeration happens, the port is expected to become suspended after 1 second at which point only 2.5mA is available.

Enumeration involves the host asking the gadget (in this case, our mobile Linux device) about its configuration options, and then requesting that some specific configuration be activated. More details on enumeration can be found in an earlier article on [the USB composite framework](<https://lwn.net/Articles/395712/>). A configuration includes the amount of current that will be required, which may be close to zero for a separately powered device, or may be the maximum supported for something that is power hungry and fully bus-powered. The host knows what it can provide and will ignore any configuration that requires too much power.

This protocol is quite suitable for a gadget that is dependent on bus power and needs a certain amount of current or else it cannot reliably function. It is less suitable for a battery-powered gadget like a smartphone that can function with no bus-power at all, but would be happy to receive as much as is available. Such a device can present two (or more) distinct configurations to the host: one that claims to require 500mA and a second one that requires zero. A host with power to spare should activate the first one. A host that cannot provide this power should reject the first and accept the second.

#### Current from other port types

There are a variety of other port types that a USB gadget can find itself connected to. A Dedicated Charging Port (DCP) provides power, at least 500mA, but no data. A Charging Downstream Port (CDP) provides data access much like an SDP, but also provides at least 1.5A, and as much as 5A, that is available even before bus enumeration. USB C-series connectors introduce more options with the same maximum current of 5A, though there is the possibility of increasing the voltage up to 20V, which would yield 100W of power.

For USB-OTG there is an extra port type, the Accessory Charger Adapter (ACA) as well as an extended form: the ACA-Dock. The ACA switches the OTG port into host mode, but also provides power to it, rather than requiring power from it. The ACA-Dock provides power and can switch the OTG port between host and gadget mode, presumably based on what else is plugged into the dock. An ACA-Dock will provide at least 1.5A, while a simple ACA can provide as little as 500mA.

Each of these peers can be detected directly by the USB [PHY](<https://en.wikipedia.org/wiki/PHY_\(chip\)>) — the circuitry responsible for the physical interface with the bus. This detection happens without needing to enter enumeration negotiations, so if power is available it can be accessed quickly.

USB connection negotiations between a host and a gadget start with the host providing a 5V level on VBUS and the gadget PHY detecting this voltage. The PHY then advertises its existence to the host by pulling one of DP or DM up to the same level as VBUS, the choice of pin giving some indication of supported bus speed. At this point the host starts enumeration. Before it applies the full VBUS voltage, the PHY can send other signals at a lower voltage and check the response. The simplest case involves setting a low voltage (e.g. 2V) on DP and checking if it is echoed back on DM. Seeing the same 2V on DM strongly implies that the DP and DM lines are shorted together, which is how a dedicated charger (DCP) can be detected. A similar but more complex signalling will detect a CDP.

For a USB-OTG port, the ID pin is supplied with 5V and, when there is no cable plugged in or when a normal 4-pin B-series plug is in place, it will stay at 5V drawing no current. As mentioned, an OTG cable will pull this pin down to GND and if a resistor is used to pull it down the resistance indicates the type of device. If ID is a short-circuit to GND, or shows a resistance of at most 1kOhm, then a simple gadget that doesn't provide any power is being attached. The OTG must provide 5V to VBUS and cannot expect anything back. If a resistance is measured between 1kOhm and 200kOhm (the minimum to be considered open-circuit), then some sort of ACA is implied and the specific resistance indicates the type of ACA.

The USB PHY controller in the mobile device can usually perform these various tests in hardware, possibly with some software support, the moment that a voltage change is detected, and can make the results available to the Linux driver. The Linux driver then just needs to tell the power-management driver how much current to draw.

#### A range of options

When connected to an SDP, and after a configuration has been negotiated, a simple number is available to the Linux device so it knows how much current it can draw — the number that was requested in the configuration that was activated. In theory, it should be safe to draw that much current at 5V. When attached to other port types, it isn't quite so simple.

According to BC-1.2, the current provided by a dedicated charger, DCP, is IDCP, which has a minimum of 500mA and a maximum of 5A. Similarly, a CDP provides ICDP, which ranges from 1.5A to 5A. ACA configurations have ranges too, the lower ends of which were mentioned earlier. Setting a current limiter in the portable device to a single number is normally quite simple. If we only have a range of allowable values, it isn't immediately clear what number we should use.

The intended meaning of these ranges is that the power source must provide at least the minimum listed current at a voltage within 5% of the 5V target, so at least 4.75V. Demands for higher current may cause the voltage to drop and, at some point, the supply may cut out completely. The supply should not cut out while the current is below the maximum, unless the voltage has first dropped below 2V.

[ ![\[Power response curves\]](https://static.lwn.net/images/2016/usb-prc-sm.png) ](<https://lwn.net/Articles/693502/>)

The graph at right (click for larger, animated version) shows hypothetical power curves that could be presented by chargers, which would be within specifications. As the load increases, the current provided increases, and then eventually the voltage starts to drop. The key requirements of the specification are that the trace must never enter the grey areas and that the charger must not shut down (depicted by a black disk) before the trace enters a pink area. [This blog post](<http://www.righto.com/2012/10/a-dozen-usb-chargers-in-lab-apple-is.html#power_curve>) shows some power curves presented by real devices. 

Pulling all of this together, the negotiations on the bus will provide an upper and lower bound for the amount of current that can be drawn. The device can safely request the minimum and if that is less than the maximum it can then slowly increase the requested current until the maximum is reached, or until the supplied voltage drops below some device-specific threshold that must be greater than 2V and at most 4.75V. Each voltage level should be averaged over 250ms to avoid being confused by transients.

While this procedure should be safe, it is not unheard of for charging hardware to be less than perfect, and some cables can introduce more resistance than one would like. So a truly cautious driver would start requesting current well below the negotiated minimum and advance slowly with large margins for error.

#### Dead batteries — when we don't have the luxury of time

One situation that is particularly difficult to handle with USB charging is that of a dead battery. If a device had a dedicated power supply, with known characteristics, and, in particular, if it was certain of a supply of a few hundred mA, then the hardware could be designed to boot with only external power — even if there is no battery. With USB there is no such guarantee. The only guarantee that USB provides without any negotiation is that 100mA can be used for up to 1 second, but longer term only 2.5mA is certain to be available. 100mA is barely enough power to boot Linux on current hardware, 2.5mA is certainly insufficient.

One option is to simply ignore the specification and pull 100mA anyway. Many chargers will support this and those that don't should just shut down, which is no worse than trying to survive with 2.5mA. There is, however, a better option.

BC-1.2 defines a Dead Battery Provision (DBP) which allows for an extremely simple handshake to request that 100mA be available for a longer term — up to 45 minutes. If the gadget device places a voltage of 0.5V on the DP pin, this will keep the bus from entering the suspended mode, so the drop to 2.5mA never happens. When making use of the DBP, the device is strongly encouraged to limit its activities to only those which are required to charge the battery to the "Weak Battery Threshold", and then activate just enough hardware to properly respond to USB enumeration and negotiate available power correctly.

To proceed correctly requires hardware that knows to set DP appropriately when the battery is flat, and software to avoid turning on any unnecessary components until the power state has been properly established.

#### The task at hand

To summarize, the tasks that Linux must support for full compliance are:

  * find out from the USB PHY what type of cable is attached and report this to the battery charger 
  * advertise USB gadget configurations with appropriate power demands 
  * determine which gadget configuration was chosen and report the available power to the battery charger 
  * adjust current within the given range to maintain suitable voltage
  * detect when the power supply is questionable during boot and limit activation of components until that is resolved

In the concluding article, we will have a look at the various device types within Linux that are responsible for these different tasks and at how they work together to produce a charging solution.
