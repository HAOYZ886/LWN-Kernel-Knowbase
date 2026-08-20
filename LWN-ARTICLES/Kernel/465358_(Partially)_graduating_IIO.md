---
title: (Partially) graduating IIO
url: https://lwn.net/Articles/465358/
date: "November 2, 2011"
category: "Device drivers-Industrial IO; Industrial IO devices"
author: "By Jonathan Corbet November 2, 2011"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
November 2, 2011

The industrial I/O (IIO) subsystem has lived in the staging tree for some time. It provides a framework for drivers that deal with all kinds of sensors that measure quantities like voltages, temperatures, acceleration, ambient light, and more. There has been [some disagreement](<https://lwn.net/Articles/390634/>) over the years about how sensors of this type should fit into the kernel; IIO, it is hoped, will provide the answer. 

The core IIO code sat out of tree for a long time; the state of the code, it is said, reflected that fact. There has been a determined effort to improve things in the staging tree, with some measurable results. There is now [a set of core IIO patches](<https://lwn.net/Articles/463814/>) that, according to maintainer Jonathan Cameron, is now ready to move out of staging and into the mainline proper. 

IIO sensors vary a lot, from simple, low-bandwidth sensors to complex, high-bandwidth devices. The initial IIO move is aimed at the first set. For this kind of sensor, the user-space interface is expected to live entirely in sysfs, under `/sys/bus/iio/devices`. Each device entry will have a number of attributes; some, like `name` and `sampling_frequency`, will be present for all sensors. Others will depend on what the sensor actually measures; the [proposed ABI](<https://lwn.net/Articles/465361/>) attempts to standardize the names of those attributes wherever possible. 

The plan is to get this core interface into the mainline, then to start moving the simpler (and cleaner) drivers after it. Support for more complex devices will come later. As of this writing, this code has not been pulled for 3.2, but that could yet happen. Meanwhile, vast numbers of IIO changes have gone into the staging tree for 3.2; there is clearly a lot of interest in getting this subsystem into shape.
