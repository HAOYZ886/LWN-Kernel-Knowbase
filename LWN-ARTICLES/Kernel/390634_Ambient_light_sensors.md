---
title: Ambient light sensors
url: https://lwn.net/Articles/390634/
date: "June 2, 2010"
category: "Device drivers-Industrial IO; Industrial IO devices"
author: "By Jonathan Corbet June 2, 2010"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
June 2, 2010

Ambient light sensors do exactly that: tell the system how much light currently exists in the environment. They are useful for tasks like automatically adjusting screen brightness for optimal readability. There are a few drivers for such sensors in the kernel now, but there is no standard for how those drivers should interface to user space. Andrew Morton recently [noticed this problem](<https://lwn.net/Articles/390637/>) and suggested that it should be fixed: ""This is very important! We appear to be making a big mess which we can never fix up."" 

As it happens, the developers of drivers for these sensors tried to solve this problem earlier this year. That work culminated in [a pull request](<http://linux.derkeiler.com/Mailing-Lists/Kernel/2010-03/msg00414.html>) asking Linus to accept the ambient light sensors framework into the 2.6.34 kernel. That pull never happened, though; Linus [thought](<http://linux.derkeiler.com/Mailing-Lists/Kernel/2010-03/msg01460.html>) that these sensors should just be treated as another (human) input device, and others [requested](<http://linux.derkeiler.com/Mailing-Lists/Kernel/2010-03/msg01218.html>) that it be expanded to support other types of sensors. This framework has languished ever since. 

Perhaps the light sensor framework wasn't ready, but the end result is that its developers have gotten discouraged and every driver going into the system is implementing a different, incompatible API. Other drivers are waiting for things to stabilize; Alan Cox [commented](<https://lwn.net/Articles/390639/>): ""We have some intel drivers to submit as well when sanity prevails."" It's a problem clearly requiring a solution, but it's not quite clear who will make another try at it or when that could happen.
