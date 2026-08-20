---
title: "Ghosts of Unix Past: a historical search for design patterns"
url: https://lwn.net/Articles/411845/
date: "October 27, 2010"
category: "Development model-Patterns"
author: "October 27, 2010 This article was contributed by Neil Brown"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

October 27, 2010

This article was contributed by Neil Brown

The exploration of design patterns is importantly a historical search. It is possible to tell in the present that a particular approach to design or coding works adequately in a particular situation, but to identify patterns which repeatedly work, or repeatedly fail to work, a longer term or historical perspective is needed. We benefit primarily from hindsight. 

The [previous](<http://lwn.net/Articles/336224/>) [series](<http://lwn.net/Articles/336255/>) of [articles](<http://lwn.net/Articles/336262/>) on design patterns took advantage of the development history of the Linux Kernel only implicitly, looking at the patterns that could be found it the kernel at the time with little reference to how they got there. Perspective was provided by looking at the results of multiple long-term development efforts, all included in the one code base. 

For this series we try to look for patterns which become visible only over an extended time period. As development of a system proceeds, early decisions can have consequences that were not fully appreciated when they were made. If we can find patterns relating these decisions to their outcomes, it might be hoped that a review of these patterns while making new decisions will help to avoid old mistakes or to leverage established successes. 

### Full exploitation

A very appropriate starting point for this exploration is the Ritchie and Thompson paper, published in Communications of the ACM, which introduced ["The Unix Time-Sharing System"](<http://cm.bell-labs.com/cm/cs/who/dmr/cacm.html>). In that paper the authors claimed that the success of Unix was not in "new inventions but rather in the full exploitation of a carefully selected set of fertile ideas." The importance of "careful selection" implies a historical perspective much like the one here proposed for exploring design patterns. A selection can only be made if previous experience is available which demonstrates a number of design avenues to choose between. It is to be hoped that identifying patterns would be one aspect of the care taken in that selection. 

Over four weeks we will explore four design patterns which can be traced back to that early Unix of which Ritchie and Thompson wrote, but which can be seen much more clearly from the current perspective. Unfortunately they are not all good, but both good and bad can provide valuable lessons for guiding subsequent design. 

"Full exploitation" is essentially a pattern in itself, and one we will come back to repeatedly. Whether it is applied to software development, architecture, or music composition, exploiting a good idea repeatedly can enhance the integrity and cohesion of the result and is - hopefully - a pattern that does not need further justification. That said, "full exploitation" can benefit from detailed illumination. We will gain such illumination for this, as for the other three patterns, by examining two specific examples. 

Ritchie and Thompson identified in their abstract several features of Unix which they felt were noteworthy. The first two of these will be our first two examples. Using their words: 

  1. A hierarchical file system incorporating demountable volumes, 
  2. Compatible file, device, and inter-process I/O, 

#### File Descriptors

The second of these is sometimes seen as a key hallmark of Unix and has been rephrased as "Everything is a file". However that term does the idea an injustice as it overstates the reality. Clearly everything is not a file. Some things are devices and some things are pipes and while they may share some characteristics with files, they certainly are not files. A more accurate, though less catchy, characterization would be "everything can have a file descriptor". It is the file descriptor as a unifying concept that is key to this design. It is the file descriptor that makes files, devices, and inter-process I/O compatible. 

Though files, devices and pipes are clearly different objects with different behaviors, they nonetheless have some behaviors in common and by using the same abstract handle to refer to them, those similarities can be exploited. A program or library routine that does not care about the differences does not need to know about those differences at all, and a program that does care about the differences only needs to know at the specific places where those differences are relevant. 

By taking the idea of a file descriptor and exploiting it also for serial devices, tape devices, disk devices, pipes, and so forth, Unix gained an integrity that has proved to be of lasting value. In modern Linux we also have file descriptors for network sockets, for receiving timer events and other events, and for accessing a whole range of new types of devices that were barely even thought of when Unix was first developed. This ability to keep up with ongoing development demonstrates the strength of the file-descriptor concept and is central to the value of the "full exploitation" pattern. 

As we shall see, the file descriptor concept was not exploited as fully as possibly it could have been, either initially or during ongoing development. Some of the weaknesses that we will find are in places where there was missed opportunity for full exploitation of file descriptors or related ideas, and many of the strengths are in places where file descriptors were used to enable new functionality. 

#### Single, Hierarchical namespace

The other noteworthy feature identified by Ritchie and Thompson (first in their list) was a hierarchical filesystem incorporating demountable volumes. 

There are three key aspects to this file system which are particularly significant for the present illustration. 

  1. It was hierarchical. We are so used to hierarchical namespaces today that this seems like it should be a given. However at the time it was somewhat innovative. Some contemporaneous filesystems, such as the one used in CP/M, were completely flat with no sub-directories. Others might have a fix number of levels to the hierarchy, typically two. The Unix filesystem allowed an arbitrarily deep hierarchy. 

  2. It allowed demountable volumes. While each distinct storage volume could store a separate hierarchical set of files, this separation was hidden by combining all of these file sets into a single all-encompassing hierarchy. Thus the idea of hierarchical naming was exploited not just for a single device, but across the union of all storage devices. 

  3. It contained device-special files. These are filesystem objects that provide access to devices, both character devices like modems and block devices like disk drives. Thus the hierarchical naming scheme covered not only files and directories, but also all devices. 

The design idea being fully exploited here is the hierarchical namespace. The result of exploiting it within a single storage device, across all storage devices, and providing access to devices as well as storage, is a "single namespace". This provides a uniform naming scheme to provide access to a wide variety of the objects managed by Unix. 

The most obvious area where this exploitation continued in subsequent development is the area of virtual filesystems, such as procfs and sysfs in Linux. These allowed processes and many other entities which were not strictly devices or files to appear in the same common namespace. 

Another effective exploitation is in the various autofs or auto-mount implementations which allow other objects, which are not necessarily storage, to appear in the namespace. Two examples are `/net/`_hostname_ which includes hosts on the local network into the namespace, and `/home/`_username_ which allows user names to appear. While these don't make hosts and users first-class namespace objects they are still valuable steps forward. In particular the latter removes the need for the tilde prefix supported by most shells and some editors (i.e. the mapping from `~username` to that user's home directory). By incorporating this feature directly in the namespace, the functionality becomes available to all programs. 

As with file descriptors, the hierarchical namespace concept was not exploited as fully as might have been possible so we don't really have a single namespace. Some aspects of this incompleteness are simple omissions which have since been rectified as mentioned above. However there is one area where a hierarchical namespace was kept separate, with unfortunate consequences that still aren't fully resolved today. That namespace is the namespace of devices. The device-special files used to include devices into the single namespace, while effective to some degree, are a poor second cousin to doing it properly. 

A little reflection will show that the device namespace in Unix is a hierarchical space with three or more levels. The top level distinguishes between 'block' and 'character' devices. The second level, encoded in the major device number, usually identifies the driver which manages the device. Beneath this are one or two levels encoded in bit fields of the minor number. A disk drive controller might use some bits to identify the drive and others to identify the partition on that drive. A serial device driver might identify a particular controller, and then which of several ports on that controller corresponds to a particular device. 

The device special files in Unix provide only limited access to this namespace. It can be helpful to see them as symbolic links into this alternate namespace which add some extra permission checking. However while symlinks can point to any point in the hierarchy, device special files can only point to the actual devices, so they don't provide access to the structure of the namespace. It is not possible to examine the different levels in the namespace, nor to get a 'directory listing' of all entries from some particular node in the hierarchy. 

Linux developers have made several attempts to redress this omission with initiatives such as devfs, devpts, udev, sysfs, and more recently devtmpfs. Given the variety of attempts, this is clearly a hard problem. Part of the difficulty is maintaining backward compatibility with the original Unix way of using device special files which gave, for example, stable permission setting on devices. There are doubtless other difficulties as well. 

Not only was the device hierarchy not fully accessible, it was not fully extensible. The old limit of 255 major numbers and 255 minor number has long since been extended with minimal pain. However the top level of "block or char" distinction is more deeply entrenched and harder to change. When network devices came along they didn't really fit either as "block" or "character" so, instead of being squeezed into a model where they didn't fit, network devices got their very own separate namespace which has its own separate functions for enumerating all devices, opening devices, renaming devices etc. 

So while hierarchical namespaces were certainly well exploited in the early design, they fell short of being fully exploited, and this lead to later extensions not being able to continue the exploitation fully. 

### Closing

These two examples - file descriptors and a uniform hierarchical namespace - illustrate the pattern of "full exploitation" which can be a very effective tool for building a strong design. While we can see with hindsight that neither was carried out perfectly, they both added considerable value to Unix and its successors, adequately demonstrating the value of the pattern. Whenever one is looking to add functionality it is important to ask "how can this build on what already exists rather than creating everything from scratch?" and equally "How can we make sure this is open to be built upon in the future?" 

The next article in this series will explore two more examples, examine their historical development, and extract a different pattern -- one that brings weakness rather than strength. It is a pattern that can be recognized early, but still is an easy trap for the unwary. 

### Exercises

The interested reader might like to try the following exercises to further explore some of the ideas presented in this article. There are no definitive answers, but rather the questions are starting points that might lead to interesting discoveries. 

  1. Make a list of all kernel-managed objects that can be referenced using a file descriptor, and the actions that can be effected through that file descriptor. Make another list of actions or objects which do not use a file descriptor. Explain how one such action or object could benefit by being included in a fuller exploitation of file descriptors. 

  2. Identify three distinct namespaces in Unix or Linux that are not primarily accessed through the "single namespace". For each, identify one benefit that could be realized by incorporating the namespace into the single namespace. 

  3. Identify an area of the IP protocol suite where "full exploitation" has resulted in significant simplicity, or otherwise been of benefit. 

  4. Identify a design element that was fully exploited in the NFSv2 protocol. Compare and contrast this with NFSv3 and NFSv4. 

#### Next article

[Ghosts of Unix past, part 2: Conflated designs](<https://lwn.net/Articles/412131/>)
