---
title: APIs for sensors
url: https://lwn.net/Articles/433601/
date: "March 16, 2011"
category: "Device drivers-Support APIs"
author: "By Jonathan Corbet March 16, 2011"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
March 16, 2011

Environmental sensors were, once upon a time, equipment which were only found in specialized settings like industrial process control or scientific research. They were expensive and tuned to a specific task. Increasingly, though, sensors are being attached to all kinds of devices. Mobile handsets have compasses, accelerometers, and more. Sensors for temperature, pressure, etc. are becoming increasingly common as well. The implications are fun; any Linux machine can be a versatile data collection device. 

The only problem with all of this is that the Linux kernel does not yet have an established API - either internal or to user space - for sensors. There _are_ interfaces for specific types of sensors; Video4Linux2 handles cameras, for example, and the hwmon subsystem deals with the specific class of sensors aimed at monitoring the health of the computer itself. In these areas, the interfaces are well established and interoperation is possible. For sensors which fall outside of these classes, though, there are no real rules. The outcome of this kind of situation is always the same: new devices are added with inconsistent interfaces, making life hard for application developers. 

This situation came to light (again) with the recent submission of [a pressure sensor driver](<https://lwn.net/Articles/433407/>) which was implemented as a misc device. It used the input subsystem to present its interface; Jonathan Cameron, who has been working on sensor interfaces, pointed out that the patch would not be accepted in that form. Input devices are meant for human input; since most humans do not communicate with their systems via large ambient pressure changes, this device did not fit. So the driver needs another home. The hwmon subsystem was suggested, but the pressure sensor is not really a hardware monitor, so the driver is not welcome there either. Arnd Bergmann also [does not like the use of the misc interface](<https://lwn.net/Articles/433610/>): 

I generally try to prevent people from adding more ad-hoc interfaces to drivers/misc. Anything that is called a drivers/misc driver to me must qualify as "there can't possibly be a second driver with the same semantics", otherwise it should be part of another subsystem with clear rules, or be put into its own file system. 

That leaves the industrial I/O (IIO) subsystem, which is meant "for devices that in some sense are analog to digital converters." IIO tries to handle a wide variety of sensors in some sort of standard way with support for events, higher bandwidth I/O, and more. There are quite a few drivers in the IIO subsystem now; the only problem is that the whole thing lives in the staging tree and the associated "TODO" list is reasonably long. The devices which are represented there now are not all consistent in their interface use - and the form of the desired interface is not at all clear. 

Still, putting together such an interface is [Jonathan's goal](<https://lwn.net/Articles/433612/>): 

To my mind, there will one day be a suitable 'sensors' subsystem so an important side point is to try and minimise interface changes needed to move to that (IIO or something better). Sysfs is easy to fix, so lets at least work on shared interfaces in there. Hwmon is a mature and reasonable starting point; it's where we got a lot of IIO's similar interfaces from. The trick is convincing people to consider generality and it's a hard trick to pull off. 

He adds that the interface and support for simple devices (those with slow data rates and hwmon-style sysfs interfaces) is in reasonably good shape. The question is how to get the rest of the job done. 

One alternative would be to define an essentially new IIO core which would be merged into the mainline. Individual drivers could then be worked into shape and moved over once they are ready. The problem is that this could be a long process, and that the mainline versions of the drivers might not initially have all of the functionality of their black-sheep staging cousins. That would mean more maintenance work keeping both versions of the driver working for some time. 

Still, that's [the approach that Arnd recommends](<https://lwn.net/Articles/433615/>). The move to the mainline is the last good chance to define an interface which will then need to be supported for many years. So some pain now, if used properly, may be warranted in order to make life easier in the future. Getting driver developers to buy into this idea may not be entirely easy; most of them spend the bulk of their time doing something other than writing Linux driver code and may lack the desire to move to a new interface when what they have now works. But that's almost certainly the best way forward. Now is almost certainly a good time for people with an interest in this area to help in the development of the mainline version of the IIO interface.
