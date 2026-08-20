---
title: "AXFS: a compressed, execute-in-place filesystem"
url: https://lwn.net/Articles/295545/
date: "August 26, 2008"
category: "Filesystems-Flash"
author: "By Jonathan Corbet August 26, 2008"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
August 26, 2008

Filesystems are clearly an area of high development interest at the moment; hardly a week goes by without a new filesystem release for Linux popping up on some list or other. All of this development is motivated by a number of factors, including the increasing size of storage devices and the increasing capability of solid-state storage. Beyond that, though, there is the simple fact that there is no single filesystem which is optimal for all applications. The recently-announced [AXFS filesystem](<http://lwn.net/Articles/295369/>) is a clear example of what can be done if one targets a specific use case and optimizes for that case only. 

At a first impression, AXFS seems like a simple and limited filesystem. It is, for example, read-only; the AXFS developers have made no provision for changing the filesystem after it is created. Some filesystems have a great deal of code dedicated to the creation of the optimal layout of file blocks on disk; AXFS has none of that. Instead, it has a simple format which divides the media into "regions" and, almost certainly, spreads accesses across the device. There is no journaling, no logging, no snapshots, and no multi-device volume management. 

What AXFS does provide is compressed storage using zlib. It is, clearly, aimed at embedded systems using flash-based storage. For such devices, a compressed filesystem can be built using the provided tools, then loaded into a minimal amount of flash on each device. It thus joins a number of other compressed filesystems - cramfs and squashfs, for example - provided for this sort of application. One interesting aspect of compressed, flash-oriented filesystems is their apparent ability to stay out of the mainline kernel. By posting AXFS for review on linux-kernel, developer Jared Hulbert may be trying to avoid a similar fate. 

The feature which makes AXFS different from squashfs and cramfs is its support for execute-in-place (XIP) files. Some types of flash can be mapped directly into the processor's address space. When running programs stored on that flash, copying pages of executable code from flash into main memory seems like a bit of a waste: since that code is already addressable by the processor, why not run it from the flash? Executing code directly from flash saves RAM; it also makes things faster by eliminating the need to copy those pages into RAM at page fault time. As a result, systems using XIP tend to boot more quickly, a feature which designers (and users) of embedded systems appreciate. 

Linux has had an [execute in place mechanism](<http://lwn.net/Articles/135472/>) for a few years now, but relatively few filesystems make use of it. AXFS has been designed from the beginning to facilitate XIP operation - that's its reason for existence (and the origin of the "X" in its name). 

There is an additional twist, though. One would ordinarily consider compressed storage and XIP to be mutually exclusive - there is little value in mapping compressed executable code into a process's address space. To be able to executed in place, a page of code must be stored uncompressed. What makes AXFS unique is its ability to mix compressed and uncompressed pages in the same executable file. So pages which will be frequently accessed can be stored uncompressed and executed in place. Pages with infrequently-needed code or which contain initialized data can be stored compressed to save space and uncompressed at fault time. 

This is a slick feature, but it is not of great use if one does not know which pages of an executable file are heavily enough used to justify storing them without compression. Trying to determine this information and manually pick the representation of each page seems like an error-prone exercise - not to mention one which would tend to create high employee turnover. So another method is needed. 

To that end, AXFS provides a built-in profiling mechanism. Each AXFS filesystem is represented by a virtual file under `/proc/axfs`; writing "`on`" to that file will cause AXFS to make a note of every page fault within the filesystem. Reading that file then yields spreadsheet-like output showing, for each file, how many times each page was faulted into the page cache. Using this data, it is possible to generate an AXFS filesystem image with an optimal number of compressed pages for the target system. 

Filesystems normally need a few rounds of review before they can make it into the mainline; some filesystems need rather more than that. AXFS is sufficiently simple, though, that it may find a quicker path into the kernel. So far, the comments have mostly been positive, with the biggest [complaint](<https://lwn.net/Articles/295552/>) being, perhaps, that its name is too close to that of the existing XFS filesystem. So a 2.6.28 merge for AXFS, while far from guaranteed, would appear to be not entirely out of the question.
