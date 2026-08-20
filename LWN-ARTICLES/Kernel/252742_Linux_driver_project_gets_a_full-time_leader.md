---
title: "Linux driver project gets a full-time leader"
url: https://lwn.net/Articles/252742/
date: "October 3, 2007"
category: Device drivers
author: "By Jake Edge October 3, 2007"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
October 3, 2007

The Linux Driver Project (LDP) just got a big boost, courtesy of Novell and Greg Kroah-Hartman. The project was [announced](<http://www.kroah.com/log/linux/free_drivers.html>) last January to much acclaim, but has languished since, buried under Kroah-Hartman's "day job" and Linux kernel development work. Now, he will be able to devote much more time to the project as his employer, Novell, has shifted his job responsibilities to [work full-time on the LDP](<http://driverdev.linuxdriverproject.org/pipermail/devel/2007-September/000001.html>). 

The original project announcement was released in conjunction with the Freedom HEC conference and was described by Kroah-Hartman as a "lame publicity stunt", because it just reiterated the standard Linux driver development model: with some hardware and some information, a driver for your device will be written. There was a new wrinkle, though; an arrangement worked out with the Linux Foundation to allow driver developers to sign non-disclosure agreements (NDAs) with hardware vendors to get access to documentation and other information about the device. NDAs for driver development are [controversial in some quarters](<http://lwn.net/Articles/203562/>), but are often required by hardware vendors. 

There are numerous benefits for Linux: the drivers will all be licensed under the GPL, will get merged into the mainline tree, and be available for all Linux users. Other free operating systems may be able to use the source to write drivers for their systems, as well. Kroah-Hartman notes that a surprising added benefit is for new kernel developers: 

Another wonderful benefit that I never had imagined in the beginning is that we are now providing a way for developers who want to write something "real" to have a place to go. The biggest response I got from my first announcement was from developers wanting to help out. I had over 80 people sign up to help out as they wanted to be able to contribute to Linux, but did not previously know how to do so in a tangible manner. This project gives them a place where they can develop and maintain a real driver for the kernel community. 

Now that he has time to devote to LDP, Kroah-Hartman has put together two [mailing lists](<http://driverdev.linuxdriverproject.org/mailman/listinfo>) along with a [wiki](<http://linuxdriverproject.org/twiki/bin/view>) to track the project. There is a mailing list for each of the two main roles, developers and project managers. The role of a project manager will be to facilitate the development, making sure that the driver hacker has what they need to get the job done and keeping the company, for whom the driver is being written, informed of the status. In short, they will shepherd an individual driver in much the same way that Kroah-Hartman is coordinating the LDP as a whole. 

In less than a week since the project restart, there are five [driver projects up for grabs](<http://driverdev.linuxdriverproject.org/pipermail/devel/2007-October/000046.html>), including a "clean-up and get merged" project that would be suitable as a first driver for someone just starting out in Linux driver development. Project managers are lining up to take on the drivers as well. The numbers of volunteers have grown, but as Kroah-Hartman notes, publicity is something the project still needs: 

We currently have over 200 people signed up to be a developer, so we doing quite well there. We also have over 25 people signed up to be a project manager, so I think we are good there too. What we do need the most help on right now is to find more companies that need our help. Spreading the word that this service is available and open to any company is the biggest importance I think at this time. 

Already, there are drivers for many different kinds of devices in the pipeline: 

[...] audio codec devices, USB timestamp devices, VOIP devices, video camera devices, lots of different types of data acquisition devices, as well as some custom bus interconnects and even some whole system-on-a-chip devices. 

Kroah-Hartman plans to reconnect with various companies that contacted him since January, but fell by the wayside. As that happens and the word gets out about the project, there should be driver development projects suitable for a wide range of interests and various levels of kernel experience. By providing a self-contained project that is targeted at inclusion in the mainline, more developers will be exposed to that process, which should expand the ranks of kernel hackers. 

Linux already supports more hardware than any operating system has before and the LDP will only extend that lead. There are huge benefits for Linux, the developers and project managers, the companies whose devices will be supported, as well as for distributors like Novell. There may be complaints about signing NDAs, but the drivers will be free, not obfuscated; once companies see how easy it is to get a high-quality driver into the kernel, they will certainly come back for more. This can only be a good thing for all free software systems, not just Linux.
