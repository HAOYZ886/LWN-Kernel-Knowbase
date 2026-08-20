---
title: Asynchronous crypto
url: https://lwn.net/Articles/109475/
date: "November 3, 2004"
category: Cryptography
author: 
---

The 2.5 development series included the addition of the kernel crypto API. This interface was added to enable in-kernel code to use cryptographic functions where needed; the IPSec code was one of its first users. This API has been extended since its addition, and it now supports a wide variety of cryptographic algorithms. 

There is just one little problem, however: the current Linux crypto API is a synchronous interface. When kernel code requests that a transformation be applied to a block of data, that work is done immediately, with a status value returned to the caller. A synchronous interface works fine when the cryptographic transformations are implemented in software. If the CPU has to do the work anyway, there is usually no time like the present to get it done. 

Increasingly, however, computers are being equipped with hardware cryptographic capabilities. It would be nice if Linux could make use of crypto hardware, especially on systems (such as high-bandwidth servers) which may have to do a lot of transformations. Hardware crypto complicates the situation, however; hardware operations take time. A synchronous interface does not work well when hardware is involved; the kernel needs to be able to go off and do other things while the hardware works through the data. Scheduling issues come into play as well; if a system has multiple crypto cards installed, it would be nice to balance the load across them and keep them all busy. 

The current crypto API does not address hardware-related issues at all. This shortcoming has been understood from the beginning; the initial crypto API deliberately did not set out to solve the entire problem. Hardware support was one of those "we'll get to that later" items. 

Evgeniy Polyakov, based in Russia, has gotten around to it with his posting of an [asynchronous crypto layer patch](<https://lwn.net/Articles/109190/>). This large patch creates a new cryptographic API which addresses the needs of hardware cryptography. There is a callback-based asynchronous interface which enables the queueing of transformation requests and notification of their completion. The patch not only includes load balancing; it also has a pluggable mechanism allowing a choice of which load balancer to use. There is a priority mechanism built in, and a failover handler which does the right thing when a cryptographic peripheral fails. There is even a request routing feature for complicated transformations (encryption followed by signing, say) which may have to be performed by a series of devices. 

The new code has been welcomed, though the developers have a number of issues with the specifics of the implementation. Chances are that those issues can be overcome, and the new asynchronous API will eventually find its way into the mainline. At that point, it will almost certainly obsolete the existing crypto APIs - for both crypto users and the implementation of software transforms. A certain amount of scrambling will be required to make everything work again, but, when the dust settles, Linux should have a much more comprehensive and capable cryptographic subsystem.
