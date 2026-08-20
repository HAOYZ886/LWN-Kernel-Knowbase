---
title: Greybus
url: https://lwn.net/Articles/648400/
date: "June 17, 2015"
category: Greybus
author: "By Jonathan Corbet June 17, 2015 LinuxCon Japan"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
June 17, 2015

* * *

[LinuxCon Japan](<https://lwn.net/Archives/ConferenceByYear/#2015-LinuxCon_Japan>)

A little while back, Greg Kroah-Hartman was given an opportunity to work on an interesting problem. [Project Ara](<http://www.projectara.com/>) is developing a phone handset that can be assembled (and customized) from a range of components; these components include speakers, displays, cameras, interesting sensors, and more. Making this work requires an internal bus that can handle devices that may come and go at any time — something that cellphones have not needed up to this point. Greg's job was to help to bring that bus into being; the result was "Greybus," which was the subject of Greg's LinuxCon Japan talk. 

Project Ara imposes some interesting requirements on its internal bus. Beyond being dynamic, the bus must be routable and secure; in other words, any two components on the bus must be able to communicate directly with each other, without any other component being able to listen in. As a result, standard buses like USB and PCIe are not suitable. After some searching, Greg and company came across the [UniPro](<https://en.wikipedia.org/wiki/UniPro>) bus, which seems to fit the bill. 

UniPro has its origins in Nokia, which wanted a way to be able to easily integrate cameras from any vendor. Google then picked it up and made it routable. At this point, Greg said, it is fast and mature. It promises low latency, has low-power modes and quality-of-service features, and [![\[Greg Kroah-Hartman\]](https://static.lwn.net/images/conf/2015/lcj/GregKroahHartman-sm.jpg)](<https://lwn.net/Articles/648409/>) promises in-order message delivery. The [standards](<http://mipi.org/specifications/unipro-specifications>) have been out there for a while; they are driven by the [MIPI Alliance](<http://mipi.org/>). 

Everything on UniPro happens by way of bidirectional connections that pass directly from one component to another; data does not pass through the processor. A connection is represented by a "CPort," which looks a lot like a network socket. There is a switch on the bus that sets up the actual routes. Messages can pass at a rate of around 10Gb/s; the bus also has message prioritization, error handling, and notification of delivery problems. What UniPro does _not_ do is streams or multicast delivery; Greg suggested the latter was a good non-feature, since it prevents modules from sniffing unrelated traffic. 

UniPro adheres to the OSI network model, except that it has no application layer defined. So the Project Ara developers made their own, which they called Greybus. (The name evidently comes from the gray color of the original prototype device; nobody has since come up with a better one). Greybus adds device discovery; all modules on the bus are self-describing. There are, it seems, advantages when the people who have to make the software side work get a say in how the hardware is designed. 

Greybus also adds a network routing layer internally and a set of class protocols for specific device types. This is something that USB and PCI got right, Greg said. When they adhere to the class protocols, devices like keyboards or WiFi adapters just work with no additional software development needed. This is important for Project Ara; it would like to see a lively market in third-party modules, and that is much more likely to happen if new modules simply work in existing systems. 

Each device on the bus offers a description to the rest of the system; it includes vendor and product IDs, a serial number, the protocols used, etc. Each module (a physical thing plugged into the phone) offers one or more "interfaces," each of which is a physical connection. CPort 0 on each interface controls the interface as a whole; other CPorts offer the actual functionality and will be what the kernel normally talks to. Those CPorts support "operations," which are a way of getting a module to do something using an interface that resembles a remote procedure call API. 

Currently a number of protocol classes have been implemented; these include the battery, vibrator, and near-field communications classes. Others, including audio, input, sensors, and cameras, are in progress. There are a few that have been left for later, including WiFi, Bluetooth, cellular modems, GPS receivers, lights, and the display — which, Greg suggested, will be a fair amount of work. 

Also built into Greybus is the ability to bridge other protocols. So, for example, devices speaking protocols like USB, I2C, I2S, or SPI can be driven directly over the bus. There is also support for tunneling protocols like [CSI](<http://mipi.org/specifications/camera-interface>) (for cameras) or [DSI](<http://mipi.org/specifications/display-interface>) (for displays). 

Greg concluded by noting that the code is all available in the [Greybus GitHub repository](<https://github.com/projectara/greybus>). It represents a piece of the Project Ara puzzle, but not the whole solution. In particular, making this device successful will require turning Android into a much more dynamic system; it is the same challenge that Linux distributors dealt with ten or fifteen years ago. It's a big job, but developers at Google are working on it. Once they are done (and the hardware is available), we will have stepped into a world where phone handsets are far more dynamic and configurable than they have been in the past. 

Greg let people look at his prototype handset after the talk. It consisted of a chassis into which a number of postage-stamp-sized modules could be plugged. It was a nice-looking device, though the chassis seems like it will always force the device to be a bit fatter and heavier than less configurable devices. Suffice to say your editor wants one. 

[Your editor would like to thank the Linux Foundation for supporting his travel to LinuxCon Japan.]
