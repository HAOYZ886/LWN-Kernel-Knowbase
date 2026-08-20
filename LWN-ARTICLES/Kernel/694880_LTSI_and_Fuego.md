---
title: LTSI and Fuego
url: https://lwn.net/Articles/694880/
date: "July 20, 2016"
category: "Long-term support initiative"
author: "By Jonathan Corbet July 20, 2016 LinuxCon Japan"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
July 20, 2016

* * *

[LinuxCon Japan](<https://lwn.net/Archives/ConferenceByYear/#2016-LinuxCon_Japan>)

It has now been nearly five years since Tsugikazu Shibata [announced](<https://lwn.net/Articles/464834/>) the launch of the long-term support initiative (LTSI) project. LTSI's objective is to provide extended support for specific kernel releases that can serve as a rallying point for embedded-system vendors and a means by which those vendors can get their patches upstream. At LinuxCon Japan 2016, Shibata-san provided an update on LTSI; he was followed by Tim Bird, who discussed the "Fuego" test framework that is now being used to help validate LTSI releases. 

#### An LTSI update

The core process for LTSI kernels has not changed much since the project's inception. LTSI releases are based on the long-term support releases maintained by Greg Kroah-Hartman, and are maintained for the same time period. They do, however, include a significant set of extra patches in the form of vendor-contributed features and backports from more recent kernels; they also go through more extensive testing than ordinary stable-kernel releases. 

Five years in, LTSI is seeing some significant adoption. The [Yocto](<https://www.yoctoproject.org/>) meta-distribution has had an option to use LTSI kernels since 2012. The [Automotive Grade Linux](<https://www.automotivelinux.org/>) project [![\[Tsugikazu Shibata\]](https://static.lwn.net/images/conf/2016/lcj/TsugikazuShibata-sm.jpg)](<https://lwn.net/Articles/694884/>) is using LTSI kernels (via Yocto). The relatively new [Civil Infrastructure Platform](<https://www.cip-project.org/>) (CIP) project is also using LTSI, with an interesting twist. Systems based on CIP are likely to be deployed in situations where they are expected to run for a long time, so there is a need for long-term support with a different value of "long-term": 10-15 years. CIP will itself be providing that support by taking over responsibility for LTSI kernels after LTSI itself has moved on. Shibata-san noted that supporting a kernel for that long is going to be an interesting challenge; he wished the project luck. 

The LTSI release process starts with one of the regular long-term support releases. There is a four- or five-month period in which patches to this kernel are prepared; these include backports and other features that are useful to the LTSI community. That is followed by a two-month merge window in which all those patches are applied. One month of validation follows; all contributors to the LTSI kernel are expected to ensure that things work properly in the final release. This process, Shibata-san claimed, leads to the production of one of the most stable and secure kernels available. 

That said, there are concerns that the current seven-month process takes too long; the latency in the process is especially acutely felt with the 4.4 kernel, which came a bit sooner than had been expected. So the project is talking about shortening the release process this time around; there would be a two-month preparation period and a one-month merge window. The final decision on that change, it seems, will come in the near future. 

#### Fuego

One of the significant changes in the LTSI release process mentioned by Shibata-san was the adoption of a new testing framework called "Fuego." Bird used the next session to talk about Fuego and how it works. In short, Fuego is the combination of the [Jenkins](<https://jenkins.io/>) continuous-integration tool, a set of scripts, and a collection of tests, all packaged within a Docker container. 

Jenkins is used to run tests based on various triggers and collect the results. It is widely used and features hundreds of extensions to handle things like email notifications or integration with source-code management systems. The big customization that Fuego has added is to separate host [![\[Tim Bird\]](https://static.lwn.net/images/conf/2016/lcj/TimBird-sm.jpg)](<https://lwn.net/Articles/694885/>) and target configuration; testing can be directed from a host, but it runs on the specific embedded target of interest. 

There is a set of "abstraction scripts" designed to make Fuego work with any specific target board; these scripts are driven by variables describing how to interact with the board, functions to get or put files and run commands, etc. The end result is a generated script to run the actual tests. There are about fifty tests integrated into the system so far; most of those are existing tests from elsewhere, but the plan is to add a bunch of new tests as well. 

The whole system is designed to be packaged up into a Docker container. The end result should be runnable on any Linux distribution without modification. 

Fuego was designed to be easy for embedded engineers to set up and run. It comes with configurations for specific target systems, including Yocto, Buildroot, OpenWrt, and more. Various target types and transports are supported; Fuego can talk to a target using a serial port, SSH, or Android's `adb` tool, among others. It is designed to send test results to a centralized repository. The end goal is to enable the creation of a decentralized test network, allowing the testing of changes on a wide variety of hardware and getting past the "I don't have that particular board" problem. 

Future plans include the decluttering of the Jenkins interface, which is rather busy at the moment. The project would like to add handling for USB connections, making it easier to use tools like `adb` to talk to handset-like devices. More documentation and more tests are on the list, as is integration with the [kernelci.org](<https://kernelci.org/>) project. 

More users and contributors would certainly be welcome. The project is using the [ltsi-dev mailing list](<https://lists.linuxfoundation.org/mailman/listinfo/ltsi-dev>) for its communications for now; more information on the project, including pointers to the repositories, can be found on [elinux.org](<http://elinux.org/Fuego>). See [this page](<http://bird.org/fuego/FrontPage>) for more information on how to install and use Fuego. 

[Your editor would like to thank the Linux Foundation for supporting his travel to LinuxCon Japan].
