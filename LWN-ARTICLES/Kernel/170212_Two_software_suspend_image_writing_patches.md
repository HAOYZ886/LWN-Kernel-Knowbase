---
title: Two software suspend image writing patches
url: https://lwn.net/Articles/170212/
date: "February 1, 2006"
category: Software suspend
author: 
---

The software suspend story seems to repeat itself endlessly. Developers debate multiple implementations while no decision gets made and software suspend in Linux continues to fall short of what it could really be. One place where this discussion might actually come to a head soon is in the storage and retrieval of the suspend image - the copy of system memory which is stored on disk while the system is down. Two approaches are being pushed; they reveal two very different views of the problem. 

One approach is the user-space interface, currently being developed by Rafael Wysocki. Rafael's patch is similar in spirit to the user-space patch [covered here last September](<http://lwn.net/Articles/153203/>). It no longer uses `/dev/kmem`, however; instead, it sets up a dedicated device for the software suspend operations. A user-space program can then invoke a set of `ioctl()` operations to freeze the system, allocate swap space, and move memory pages to their resting place - possibly compressing or encrypting them on the way. The [documentation file](<https://lwn.net/Articles/170215/>) provided with the patch gives a good introduction to the interface and how it should be used. 

In the other corner we have Nigel Cunningham, who has recently broken out the [modules mechanism](<http://lwn.net/Articles/170200/>) from his Suspend2 patch set. Rather than move image writing and reading support to user space, this patch sets up a complex kernel interface for plugins which take on parts of that task. There are two types of plugins: "filter" plugins which transform the image data (performing encryption, say) and "writer" plugins which handle the actual storage I/O. Parts of the code anticipate "misc" and "checksum" plugins as well, but those are not currently supported. 

The [plugin API](<https://lwn.net/Articles/170221/>) is somewhat complex. Each plugin has eleven methods to provide to the core suspend code; these handle memory allocation, configuration, initialization and cleanup. Filter plugins must define three more methods to handle data passing through for processing. And writer modules have an additional 21 methods to provide for dealing with various parts of that task. There are, it seems, a lot of things that have to be done to get an image written to (and read from) persistent storage. 

The two patches are clearly incompatible - there is no point in setting up an elaborate in-kernel interface if the whole process is to be moved out of the kernel altogether, and vice-versa. So, before merging either of these patches, somebody will have to make a decision. Anyone looking for tea leaves to read might take a hint from the fact that the user-space patches are currently in the -mm tree. As the reiser4 folks (among others) know, however, the road from -mm to mainline can be long and perilous.
