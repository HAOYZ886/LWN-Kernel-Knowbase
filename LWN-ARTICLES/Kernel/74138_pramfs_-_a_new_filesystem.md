---
title: "pramfs - a new filesystem"
url: https://lwn.net/Articles/74138/
date: "March 3, 2004"
category: Filesystems
author: 
---

Steve Longerbeam (of MontaVista) has sent out [an announcement](<http://lwn.net/Articles/74122/>) for a new filesystem called "pramfs." He would like to see pramfs merged into the mainline kernel in the near future; let it not be said that embedded Linux companies do not contribute to the kernel. 

Pramfs (the "protected and persistent RAM special filesystem") is a specialized filesystem; it is intended for use in embedded systems which provide a bank of non-volatile memory for user data storage. Think, for example, of a phone book housed within a mobile telephone. Such memory tends to be fast, but it is not normally part of the system's regular core memory. It also tends to be important; cell phone users will not tolerate a phone which scrambles their phone numbers. 

To meet the special needs presented by non-volatile RAM filesystems, pramfs does a number of things differently than normal filesystems. Since there is no need to worry about the (nonexistent) performance impacts of block positioning, pramfs doesn't. Since pramfs filesystems are expected to live in fast memory, there is generally no performance benefit to caching pages in main memory. So pramfs, interestingly, forces all file I/O to be direct; essentially, it forces the `O_DIRECT` flag on all file opens. In that way, pramfs gets the benefits of shorting out the page cache without having to change applications to use `O_DIRECT` explicitly. 

Pramfs also goes out of its way to avoid corruption of the filesystem. If the underlying non-volatile RAM is represented in the system's page tables, it is marked read-only to keep a stray write from trashing things. When an explicit write to the filesystem is performed, the page permissions are changed only for the time required to perform the I/O. Pramfs disallows writes from the page cache; one practical result of that prohibition is that shared mappings of pramfs-hosted files are not possible. 

See [the pramfs web site](<http://pramfs.sourceforge.net/>) for more information.
