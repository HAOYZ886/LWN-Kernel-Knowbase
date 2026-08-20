---
title: "LCA: The state of the Nouveau project"
url: https://lwn.net/Articles/217866/
date: "January 17, 2007"
category: "Device drivers-Nouveau; Nouveau; Reverse engineering"
author: 
---

In any conference, there comes a time when one has to wonder what the people who do the talk scheduling were thinking. For lca2007, that moment came when your editor realized that the talks on OLPC (Jim Gettys), real [![\[Dave Airlie\]](https://static.lwn.net/images/conf/lca2007/Airlie-sm.jpg)](<https://lwn.net/Articles/217869/>) time (Ted Ts'o), and Nouveau were all scheduled together. Nouveau won out, but it was not an easy decision. 

The [Nouveau project](<http://nouveau.freedesktop.org/>) is an effort to develop a set of free 3D drivers for NVidia chipsets. NVidia has long annoyed the free software community with its refusal to release free drivers or programming information for its video chipsets. The Nouveau folks have had enough of that, and they are doing something about it. Dave Airlie used his slot at linux.conf.au to talk about the project and its current status. 

Nouveau got its start in February 2005, though serious work did not begin until June of that year. The project was announced at FOSDEM 2006, at which point others started to help. There are currently about six developers doing serious work on Nouveau. 

The project is relying on reverse engineering for the information needed to write free drivers. To that end, the developers have put together [a set of tools](<http://dri.freedesktop.org/wiki/ReverseEngineering>). At the top of the list is renouveau, which is designed to reveal the commands sent to the card in response to specific operations. Using the existing binary drivers, renouveau sets up a context, then scans the process's mappings until it finds the command FIFO. It then requests an operation and sees how the FIFO changes. With enough operations, a pretty good idea of how the adapter is programmed to specific ends can be had. This was not a trivial tool, and the better part of a year was put into its development. 

Renouveau is useful for examining the FIFO, but it doesn't help with reads and writes to I/O registers. For that, there's another set of tools, starting with valgrind-mmt - a version of valgrind designed to trap I/O memory operations. Libsegfault is a modified version of `mmap()` which doesn't actually do the mappings as the caller would like; it traps the subsequent segmentation faults and dumps out the operations. There is another tool, called kmmio, which performs a similar task for register operations done in kernel space. Finally, the project uses a BIOS tracer which runs BIOS code in x86emu and traps I/O register accesses. 

All of the information obtained from these tools is supplemented with hints from the old, free nv driver. There is also, says Dave, information "which shouldn't be there" to be found on some Russian web sites. 

Where has all of this information led the project? Basic tasks, like the allocation of instance RAM and FIFO initialization, are working. Hardware context switching works - on little-endian machines. There is 2D support derived from the nv driver; it offers basic EXA and RandR 1.2 support. On the 3D front, the Mesa TCL (transform, clipping, and lighting) driver mostly works. Textures and objects do not, however. It is possible to run glxgears on nv4x chips. It has taken some time to get to this point, but Dave thinks that things will start to move a lot faster from here. 

The next milestone would be to run Quake 3. That is, says Dave, an obligatory step on the roadmap. Getting there will involve texture support, a better memory manager, and better locking in the kernel DRM code. The developers (Dave in particular) are aiming for RandR 1.2 multi-head support. Once all of this is in place, the nouveau driver will have reached a reasonably capable state. 

There are a lot of people asking when this will be; Dave says that the project's IRC channel is often overwhelmed by spectators looking for news. There is no wish to push the code out ahead of its time; among other things, that would nail down the API between the kernel and the X server, making things harder to change. The current hope is to have some sort alpha release toward the end of 2007. 

For people wanting to help, Dave had a simple message: they need developers. There's not much for people who can't work on driver code to do at this point. Graphics drivers, he says, are not as hard as people think. Finally, he addressed the issue of the $10K pledge for the project. It rather took the developers by surprise; they had not endorsed this drive, and had held some doubts as to whether it would be successful. How the pledge money will be handled is still being worked out; it looks like it will mostly be used for hardware purchases. 

Lack of support for 3D video adapters has stalled the community for years; there has been a long wait in the hope that the vendors would come to their senses. That wait is just about over. The Nouveau project (along with various others) shows that we have the resources to figure out how our hardware works, even in the face of complex devices and uncooperative vendors. It would be better if we did not have to take things into our own hands this way, but it is nice to see how well we can do it when the need arises.
