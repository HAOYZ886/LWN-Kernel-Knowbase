---
title: The unveiling of kdbus
url: https://lwn.net/Articles/580194/
date: "January 13, 2014"
category: Message passing; kdbus
author: "By Jonathan Corbet January 13, 2014 linux.conf.au 2014"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
January 13, 2014

* * *

[linux.conf.au 2014](<https://lwn.net/Archives/ConferenceByYear/#2014-linux.conf.au>)

Sporting an "Open Source Tea Party" T-shirt, Lennart Poettering used his [linux.conf.au](<http://linux.conf.au>) talk to introduce an effort that he and several others have been working on for the better part of the last year: reimplementing the [D-Bus](<http://www.freedesktop.org/wiki/Software/dbus/>) mechanism within the kernel. The result, should it make it through the review process, will equip Linux with a proper native interprocess communication mechanism for, Lennart said, the first time ever. 

#### The good and bad of D-Bus

Unlike most other kernels, Linux has never had a well-designed IPC mechanism. Windows and Mac OS have this feature; even Android, based on Linux, has one in the form of the "binder" subsystem. Linux, instead, has only had the primitives — sockets, FIFOs, and shared memory — but those have never been knitted together into a reasonable application-level API. Kdbus is an attempt to do that knitting and create something that is at least as good as the mechanisms found on other systems. 

Linux does have D-Bus, which he said, is a powerful IPC system; it is the closest thing to a standard in this area as can be found on Linux. Lennart put up an extensive list of advantages to using D-Bus. It provides a nice method-call transaction mechanism (allowing for sending a message and getting a response) and a means for sending "signals" (notifications) to the rest of the system. There is a discovery mechanism to see what else is running on the bus and the introspection facilities needed to learn about [![\[Lennart
Poettering\]](https://static.lwn.net/images/conf/2014/lca/LennartPoettering-sm.jpg)](<https://lwn.net/Articles/580196/>) what services are offered. D-Bus includes a mechanism for the enforcement of security policies, a way of starting services when they are first used, type-safe marshaling of data structures, and passing of credentials and file descriptors over the bus. There are bindings for a wide range of languages and network transparency as well. 

On the other hand, D-Bus also suffers from a number of limitations. It is well suited to control tasks, but less so for anything that has to carry significant amounts of data. So, for example, D-Bus works well to tell a sound server to change the volume, but one would not want to try to send the actual audio data over the bus. The problem here is the fundamental inefficiencies of the user-space D-Bus implementation; a call-return message requires ten message copies, four message validations, and four context switches — not the way to get good performance. Beyond that, credential passing is limited, there are no timestamps on messages, D-Bus is not available at early boot, connections to security frameworks (e.g. SELinux) must happen in user space, and there are race conditions around the activation of services. D-Bus also suffers from what Lennart described as a "baroque code base" and heavy use of XML. 

Even so, Lennart said, D-Bus is "fantastic" and it solves a number of real problems. Ten years of use have shown that the core design is sound. It is also well established and widely used. So the right thing to do is not to replace D-Bus, but to come up with a better implementation. 

#### Into the kernel

That implementation is kdbus, an in-kernel implementation of D-Bus. This implementation is able to carry large amounts of data; it can be reasonably used for gigabyte-sized message streams. It can perform zero-copy message passing, but even in the worst case, a message and its response are passed with no more than two copy operations, two validations, and two context switches. Full credential information (user ID, process ID, SELinux label, control group information, capabilities, and much more) is passed with each message, and all messages carry timestamps. Kdbus is always available to the system (no need to wait for the D-Bus daemon to be started), Linux security modules can hook into it directly, various race conditions have been fixed, and the API has simplified. 

Kdbus is implemented as a character device in the kernel; processes wishing to join the bus open the device, then call `mmap()` to map a message-passing area into their address space. Messages are assembled in this area then handed to the kernel for transport; it is a simple matter for the kernel to copy the message from one process's mapped area to another process's area. Messages can include timeouts ("method call windows") by which a reply must be received. There is a name registry that is quite similar to the traditional D-Bus registry. 

The "memfd" mechanism enables zero-copy message passing in kdbus. A memfd is simply a region of memory with a file descriptor attached to it; it operates similarly to a memory-mapped temporary file, "but also very differently." A memfd can be "sealed," after which the owning process can no longer change its contents. A process wishing to send a message will build it in the memfd area, seal it, then pass it to kdbus for transport. Depending on the size of the message, the relevant pages may just be mapped into the receiving process's address space, avoiding a copy of the data. But the break-even point is larger than one might expect; Lennart said that it works better to simply copy anything that is less than about 512KB. Below that size, the memory-mapping overhead exceeds the savings from not copying the data. 

Memfds can be reused at will. A process that needs to repeatedly play the same sound can seal the sample data into a memfd once and send it to the audio server whenever it needs to be played. All told, Lennart said, memfds work a lot like the Android "ashmem" subsystem. 

The signal broadcasting mechanism has been rewritten to use [Bloom filters](<https://en.wikipedia.org/wiki/Bloom_filter>) to select recipients. A Bloom filter uses a hash to allow the quick elimination of many candidate recipients, making the broadcast mechanism relatively efficient. 

There is a user-space proxy server that can be used by older code that has not been rewritten to use the new API, so everything should just work on a kdbus-enabled system with no code changes required. 

When will this code make its appearance? It has been [announced](<https://lwn.net/Articles/580195/>) on the D-Bus mailing list, and the code is available in the relevant repositories now. The main thing that is missing at the moment is the policy enforcement mechanism. Everything will work, Lennart said, if one doesn't mind that it will all be "horribly insecure." The plan is to get the code merged into the mainline kernel sometime in 2014. He is optimistic that this will work out; having Greg Kroah-Hartman involved in the process helps with his confidence there. But Lennart noted that two previous attempts to get D-Bus functionality into the kernel have failed, so there are no guarantees. Stay tuned over the course of the next year to see how it goes. 

See [kdbus.txt](<https://code.google.com/p/d-bus/source/browse/kdbus.txt>) in the kernel-side source repository for more information on the design of kdbus. 

[Your editor would like to thank linux.conf.au for funding his travel to Perth].
