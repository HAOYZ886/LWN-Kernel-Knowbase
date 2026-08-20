---
title: Into the ABISS
url: https://lwn.net/Articles/110277/
date: "November 9, 2004"
category: "Block layer-IO scheduling; Elevator; IO scheduler"
author: 
---

[Version 2 of the Active Block I/O Scheduling System (ABISS)](<http://lwn.net/Articles/110267/>) was released on November 9\. At a first glance, ABISS looks like yet another I/O scheduler for a kernel which already has a few of them - and that it is. But there is more to ABISS which makes it worth a look. 

The goal behind ABISS is to enable applications to request (and receive) a guaranteed I/O rate to a specific file. It is implementing a sort of isochronous stream capability for the Linux block layer. The target applications might be multimedia recording and playback programs, or, perhaps, some sort of data acquisition system. Any application which needs assurance that it can transfer data to or from the filesystem at a given rate could benefit from ABISS. 

For now, guaranteed data rates are only supported for read access, and only for a few filesystems. The core of the read side of ABISS is the "playout buffer." It is, for all practical purposes, a circular buffer in kernel space which is filled at the requested I/O rate. As long as the application does not exceed its requested rate for long periods of time, the data it requests should always be located in the buffer, and thus immediately available. The playout buffer is integrated with the page cache, so accessing the file via `mmap()` will also work - though, in that case, the application must inform ABISS of its progress through the file so that playout buffer pages can be released when no longer needed. 

Setting up this buffer requires a few steps. The application uses an `ioctl()` call to request a guaranteed read rate; that request is then passed back to a user-space daemon for approval. The daemon is supposed to keep track of all such requests and ensure that the system actually has enough resources to implement another fixed-rate stream. Any policy decisions on which processes are allowed to request guaranteed-rate behavior - and the rates they can ask for - are also made in the user-space daemon. 

If the daemon approves the request, the kernel builds an in-memory map of the location of the file's data blocks. This map is used when filling the playout buffer; its real purpose is to do the file location lookup work ahead of time and minimize unexpected I/O while the file is being processed. The operational phase consists of filling the playout buffer at the given rate while not allowing it to get too large. The idea is conceptually simple, though the actual implementation involves a number of somewhat tricky details. 

ABISS differs from other I/O schedulers in that it does not just fit neatly into the block layer. Each filesystem must have ABISS support explicitly added to it. In particular, ABISS must be able to intercept `ioctl()` calls and, build the location map. When the filesystem-level code decides to look for a specific block within the file, the ABISS code, which may already have that location in its map, needs a chance to short out the usual lookup code. Finally, ABISS must be notified when a file is truncated, since it needs to adjust the location map to match the new size. Since filesystem-level changes are needed, ABISS does not support all filesystems in the Linux kernel; version 2 only works with FAT, VFAT, and ext3. 

Underneath it all is a real I/O scheduler. The primary feature there is the implementation of I/O request priorities. Requests to fill the playout buffer go in at a high priority, and will be executed before most others. The ABISS I/O scheduler also implements a set of "best effort" priorities which can be used when guaranteed I/O rates are not required. 

More information can be found on the [ABISS project page](<http://abiss.sourceforge.net/>).
