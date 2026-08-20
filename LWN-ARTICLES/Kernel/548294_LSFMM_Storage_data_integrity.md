---
title: "LSFMM: Storage data integrity"
url: https://lwn.net/Articles/548294/
date: "April 24, 2013"
category: Data integrity
author: "By Jake Edge April 24, 2013 LSFMM Summit 2013"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
April 24, 2013

* * *

[LSFMM Summit 2013](<https://lwn.net/Articles/LSFMM2013/>)

In a combined Storage and Filesystem track session at the 2013 LSFMM Summit, Darrick Wong kicked off a discussion of storage data integrity protection. He gave a talk at the 2011 Linux Plumbers Conference on the same topic, so he wanted to update the assembled developers on what had changed. There are standards for attaching metadata to data that is being written to or read from storage devices that can be used to check the data's integrity. The SCSI data integrity field (DIF) can hold a CRC to detect data corruption as well as 32-bit block numbers that can catch misplaced writes. 

There are applications that want to use DIF, but there is currently no user-space interface to do so. One way to get there might be using space in the kernel asynchronous I/O (KAIO) interface to add a pointer to "something else" that holds the protection information. Another option is to "pull a Windows trick" and have length and version information so that the kernel knows which version of the API is being used, thus how much data to copy from user space, Wong said. 

Another idea that comes up repeatedly is to use Joel Becker's batched I/O interface, called sys_dio, Wong said. That interface, which provides a way to attach integrity information to I/O operations, was originally something Becker did that was customized for Oracle's use case. Becker would like to make it more generic. It is a nicer interface that is "purely asynchronous" for direct I/O (i.e. `O_DIRECT`); Becker put out an [RFC for sys_dio](<http://permalink.gmane.org/gmane.linux.file-systems/51015>) two years ago. It was used by Martin Petersen to pass the protection information in and out of the kernel, but neither he nor Petersen has yet had time to work on finishing it up. 

With his "database hat on", Petersen said that both solutions (KAIO and sys_dio) would be useful. He went on to describe how applications use the protection information to widen the window over which the data is protected. The application can query the block layer to find out what kind of CRC to generate. Those applications (typically database systems like Oracle and MySQL) already have a block-oriented view, so they calculate the proper CRC to send with the data. If it arrives and doesn't pass the CRC test, the application may still be able to recreate it, which is why it is interested in integrity handling. The kernel can do the calculation for other applications, Petersen said. 

The SCSI T10 DIF is only concerned with protection on the path between the host adapter (HBA) and the storage device, but Petersen authored the [data integrity extensions (DIX) [PDF]](<https://oss.oracle.com/~mkp/docs/dix-draft.pdf>) to add end-to-end data integrity by including the operating system and applications. He is looking into making DIX look "less blocky" on the host, so that it could calculate a CRC on a list of scatter/gather I/O operations, then pass it to the HBA, which could then write it in the proper block format. 

But Becker was not convinced that applications need to be shielded from dealing with blocks directly. Any application that cares about end-to-end integrity will also care about the blocks on disk. Petersen would like sys_dio to not preclude byte-oriented uses, though. Wong said that it is easier to use KAIO for that case, however. 

In the end, the storage and filesystem developers agreed to look carefully at what Wong plans to post to the lists over the coming months, with an eye toward resolving these issues.
