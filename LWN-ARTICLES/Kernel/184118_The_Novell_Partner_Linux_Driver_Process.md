---
title: The Novell Partner Linux Driver Process
url: https://lwn.net/Articles/184118/
date: "May 17, 2006"
category: Device drivers
author: 
---

Every so often, somebody shows up on the linux-kernel list with the same bright idea: separate the device drivers from the rest of the kernel and release them independently. Then drivers could be installed or updated without having to change the entire kernel. This idea never gets very far; among other things, it implies the creation of a stable driver API, which is just [not in the plans](<http://www.kroah.com/log/linux/stable_api_nonsense.html>). But the idea keeps coming back anyway. 

When Novell's [breathless press release](<http://www.prnewswire.com/cgi-bin/stories.pl?ACCT=104&STORY=/www/story/05-17-2006/0004363610&EDATE=>), describing a "device driver breakthrough" which "solves Linux device driver compatibility issues," your editor's first thought was that this old idea had returned yet again. This breakthrough process "allows customers to obtain drivers independently of Novell kernel updates," after all, and is said to make life easier for vendors. As it turns out, however, Novell has no plans for defining any sort of stable kernel API; instead, it has created a mechanism making it easier for vendors to cope with the existing, dynamic API. 

Essentially, a vendor with a driver for its hardware can approach Novell, pay whatever fee is required to become a "partner," and have its driver distributed through the SUSE YaST mechanism. If the partner supplies versions of the driver which work with distributed SUSE kernels, Novell will make sure that each user gets the right version. Novell will provide API change notifications, helping vendors to keep their drivers working with current kernels. If the vendor becomes an Extra Special partner, Novell will take care of much of the driver updating work themselves. 

To some, this program looks for a way for Novell to help vendors who ship proprietary drivers. And there may be some truth to that view. But the real customer base may be elsewhere. Imagine that you are a vendor selling products into a highly competitive market. When your new widget comes out, you do the right thing and contribute a driver to the mainline tree. Even if the driver is accepted on the same day (a relatively unlikely course of events), it will not appear in a released kernel for a month or two, and it will not show up in released distributions for some months (or years) after that. By the time normal users can install the driver, the device is already obsolete and being replaced by something newer, shinier, and faster. And, in any case, having the driver in new distributions is of little help to customers who are running older kernels and don't want to change that. 

The Novell program will make it easy for this vendor to make drivers available for the range of currently installed SUSE systems, without forcing a kernel upgrade on their customers. If the program is done right, it could change the landscape for the better: vendors would have an easier time supporting the range of distributor kernels, and users would get current drivers, even on older systems. If done wrong, it could lead to more out-of-tree drivers, but Novell appears to have anticipated that concern. From the [driver partner FAQ](<http://developer.novell.com/wiki/index.php/FAQ_for_the_Partner_Linux_Driver_Process>): 

As an active member of the open source community, Novell's position is clear: The best place for partners to develop kernel drivers is upstream in the kernel.org source tree, where kernel driver code benefits from thorough review and community involvement. Novell promotes having all Linux device drivers be a part of the official kernel.org source tree. 

As long as vendors use this program as a backporting mechanism, it will do nothing but good for everybody involved. If they use it as a way to avoid the kernel development process or the need to release their code, the benefits will be rather less. The initial signs are good enough, however, that it is worth wishing Novell luck in this endeavor.
