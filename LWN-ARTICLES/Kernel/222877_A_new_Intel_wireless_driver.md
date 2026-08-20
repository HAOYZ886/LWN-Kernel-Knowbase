---
title: A new Intel wireless driver
url: https://lwn.net/Articles/222877/
date: "February 20, 2007"
category: "Device drivers-Wireless networking; Devicescape stack"
author: 
---

Almost exactly one year ago, Intel announced the [ipw3945 project](<http://lwn.net/Articles/173735/>) \- a free driver for its 3945ABG wireless adapters. This move was welcomed as a refreshing change from the usual mode of operation in the wireless area, which usually involves binary-only drivers. Even so, this driver was greeted with some complaints; in particular, the binary-only "regulatory daemon" was not a popular idea, despite the fact that it ran entirely in user space. The ipw3945 driver was never merged into the mainline kernel. 

In many cases, just getting free drivers from companies seems like a lot to ask. Getting them to go back and start over is often out of the question. That is just what Intel has done, however, and, on February 9, the [new version of the driver](<http://lwn.net/Articles/221550/>) was announced, complete with [a shiny new web site](<http://intellinuxwireless.org/iwlwifi>). The new driver should prove more popular than the old one was. 

The user-space regulatory daemon is no more. Intel's engineers, it seems, have found a way to move the regulatory function into the device's firmware, getting the host processor out of the regulatory compliance business altogether. That is probably a more robust solution in general, even though, strictly speaking, the flexibility of the hardware has been reduced. Most users will likely look at the tradeoff - better regulatory compliance and no binary-only daemon - and like what they see. Of course, those who see binary-only device firmware as an infringement of their freedom will not feel that the situation has improved much. 

Another significant change is that the new driver works with the Devicescape 802.11 stack. Devicescape remains the intended direction for wireless networking in the Linux kernel, so the new driver should be more easily integrated. At least, that will be the case once Devicescape gets into the mainline. For now, Linux users wanting to try out the new driver will also have to get a version of the d80211 module (available from the Intel site) and build that for their kernels as well. 

That leads to the obvious question: when will Devicescape make it into the mainline kernel? The process of getting that code ready for merging has taken rather longer than desired, but it is still moving forward. The [current plan](<https://lwn.net/Articles/222878/>), it seems, is to rebase the Devicescape code to 2.6.21-rc1, once that's released, and get the result included in the -mm kernel. If all goes well, the Devicescape stack might just find its way into 2.6.22. That would be a major step forward for wireless networking in Linux. 

Back to the Intel driver: one thing that is still lacking is any sort of hardware documentation. Anybody not working for Intel will be limited in what they can do with this driver by what they can learn from the code itself. Your editor asked Intel about hardware documentation; we were told: 

The reality is the driver sources are the programming information for the hardware. As time goes forward we spend some time trying to improve the comments in the headers for the source files to make it more clear what they do and to provide some overviews of theory-of-operation, but there isn't any self-contained accurate document that covers everything you need to know to program and operate the device. 

Given the choice between developing code and writing documentation, the Intel hackers went for the code.
