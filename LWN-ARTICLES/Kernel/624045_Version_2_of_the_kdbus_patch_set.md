---
title: Version 2 of the kdbus patch set
url: https://lwn.net/Articles/624045/
date: "December 3, 2014"
category: Message passing; kdbus
author: "By Jonathan Corbet December 3, 2014"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
December 3, 2014

When the long-awaited kdbus patch set hit linux-kernel at the end of October, it [ran into a number of criticisms](<https://lwn.net/Articles/619068/>) from reviewers. Some developers might have given up in discouragement, muttering about how unfriendly the kernel development community is. The kdbus developers know better than that, though. This can be seen in the [version 2 posting](<https://lwn.net/Articles/622645/>); the code has changed significantly in response to the comments that were received the first time around. Kdbus may still not be ready for immediate inclusion into the mainline, but it does seem to be getting closer. 

#### No more device files

One of the biggest complaints about the first version was its use of device files to manage interaction with the system. Devices need to be named; that forced a hierarchical global naming system on kdbus domains — which were otherwise not inherently hierarchical. The global namespace imposed a privilege requirement, making it harder for unprivileged users to create kdbus domains; it also added complications for users wanting to checkpoint and restore containers. 

The second version does away with the device abstraction, replacing it with a virtual filesystem called "kdbusfs." This filesystem will normally be mounted under `/sys/fs/kdbus`. Creating a new kdbus domain (a container that holds a namespace for one or more buses) is simply a matter of mounting an instance of this filesystem; the domain will persist until the filesystem is unmounted. No special privileges are needed to create a new domain — but mounting a filesystem still requires privileges of its own. 

A newly created domain will contain no buses at the outset. What it does have is a file called `control`; a bus can be created by opening that file and issuing a `KDBUS_CMD_BUS_MAKE` `ioctl()` command. That bus will remain in existence as long as the file descriptor for the `control` file is held open. Only one bus may be created on any given `control` file descriptor, but the `control` file can be opened multiple times to create multiple buses. The `control` file can also be used to create custom endpoints for well-known services. 

Each bus is represented by its own directory underneath the domain directory; endpoints are represented as files within the bus directory. Connecting to a bus is a matter of opening the kdbusfs file corresponding to the desired endpoint; for most clients, that will be the file simply called `bus`. Messages can then be sent and received with `ioctl()` commands on the resulting file descriptor. 

As can be seen, the device abstraction is gone, but the interface is still somewhat device-like in that it is heavily based on `ioctl()` calls. There has been a small amount of discussion on whether it might make more sense to just use operations like `read()` and `write()` to interact with kdbus, but there appears to be little interest in making (or asking for) that sort of change. 

#### Metadata issues

A significant change that has been made is in the area of security. In version 1, the recipient of a message could specify a set of credential information that must accompany the message. This information can include anything from the process ID through to capabilities, command line information, audit information, security IDs, and more. Some reviewers (Andy Lutomirski in particular) complained that this approach could lead to information leaks and, maybe, worse security issues; instead, they said, the sender of a message should be in control of the metadata that goes along with the message. 

The updated patch set contains a response to that request by changing the protocol. When a client connects to the bus, it runs the `KDBUS_CMD_HELLO` `ioctl()` command to set up a number of parameters for the connection; one of those parameters is now a bitmask describing which metadata can be sent with messages. It is possible for the creator of the bus to specify a minimum set of metadata to go with messages, though; in that case, a client refusing to send that metadata will not be allowed to connect to the bus. 

There is still some disagreement over _which_ metadata should be sent, whether it's optional or not. Andy [disagrees](<https://lwn.net/Articles/624097/>) with providing command-line (and related) information, on the basis that it can be set by the process involved and thus carries no trustworthy information. This metadata is evidently used mostly for debugging purposes; Andy suggests that it should just be grabbed out of `/proc` instead. He is also opposed to the sending of capability information, noting that capabilities are generally problematic in Linux and their use should not be encouraged. 

One other interesting bit of metadata that can be attached to messages is the time that the sending process started executing. It is there to prevent race conditions associated with the reuse of process IDs, which can happen quickly on a busy system. Andy dislikes that approach, noting that it will not work well with either namespaces or checkpointing. He prefers instead his own "[highpid](<https://lwn.net/Articles/624100/>)" solution. This patch adds a second, 64-bit, unique number associated with each process; interested programs can then detect process ID reuse by seeing if that number changes. Eric Biederman [disagreed](<https://lwn.net/Articles/624103/>) with that approach, saying ""What we need are not race free pids, but a file descriptor based process management api."" Andy [was not opposed](<https://lwn.net/Articles/624104/>) to that idea, but he would like to see something simple that can be of use to kdbus now. 

Andy had a number of other comments, including [pointing out](<https://lwn.net/Articles/624108/>) a couple of places where, he contended, he could use kdbus to gain root access on any system where it was installed. Even so, he [seems happy](<https://lwn.net/Articles/624109/>) with the direction the code is going, saying ""And thanks for addressing most of the issues. The code is starting to look much better to me."" 

#### Toward the mainline

In theory, resolving the remaining issues should be relatively straightforward, though it is not hard to see the "highpid" idea running into resistance at some point. But the number of reviewers for the second kdbus posting has been relatively small, perhaps as a result of the holidays in the US. The addition of a significant core API of this type requires more attention than kdbus has gotten so far. That suggests that there may still be significant issues that have not yet been raised by reviewers. Kdbus is getting closer to mainline inclusion, but it may well take a few more development cycles to get to a point where most developers are happy with it.
