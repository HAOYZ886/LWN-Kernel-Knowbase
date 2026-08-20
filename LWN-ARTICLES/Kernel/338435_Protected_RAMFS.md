---
title: Protected RAMFS
url: https://lwn.net/Articles/338435/
date: "June 24, 2009"
category: Filesystems
author: "June 24, 2009 This article was contributed by Goldwyn Rodrigues"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

June 24, 2009

This article was contributed by Goldwyn Rodrigues

Many embedded systems have a block of non-volatile RAM (NVRAM) separate from normal system memory. A recent patch, [posted ](<http://lwn.net/Articles/337535/>) by Marco Stornelli, is a filesystem for these kinds of NVRAM devices, where the device could store frequently accessed data (such as the address book for a cellphone). Protected RAMFS (PRAMFS) protects the NVRAM-based filesystem from errant or stray writes to the protected portion of the RAM caused by kernel bugs. Because it is stored in the NVRAM, the filesystem can survive a reboot, and hence can also be used to keep important crash information. 

#### Basic Features

PRAMFS is robust in the face of errant writes to the protected area, which could arise due to kernel bugs. The page table entries that map the backing-store RAM are marked read-only on initialization. Write operations to the filesystem temporarily mark the pages to be written as writable, the write operation is carried out with locks held, and then the pte is marked read-only again. This limits the writes to the filesystem in the window when the locks are held. The write-protection feature can be disabled by the kernel config option `CONFIG_PRAMFS_NOWP`. 

PRAMFS forces all files to use direct-IO. The `filp->f_flags` is set to `O_DIRECT` when the files are opened. Opening all files as `O_DIRECT` avoids page caching, and data is written immediately to a storage device. This is nearly equal to the speed of the system RAM, but it forces applications to do block-aligned I/O. 

PRAMFS does not have recovery facilities, such as journaling, to survive a crash or power failure during a write operation. The filesystem maintains checksums for the superblock and inode to check the validity of the stored object. An inode with an incorrect checksum is marked as bad, which may lead to data loss in case of power failure during a write operation. 

PRAMFS also supports [execute in place](<http://en.wikipedia.org/wiki/Execute_in_place>) (XIP), which is a technique that executes programs directly from the storage instead of copying it into RAM. For a RAM filesystem, XIP makes sense since the system can execute from the storage device as fast as it can from the system RAM, and it does not make a duplicate copy in RAM. 

#### Usage

There is no mkfs utility to create a PRAMFS. The filesystem is automatically created when the filesystem is mounted with the `init` option. The command to create and mount a PRAMFS is: 

```
# mount -t pramfs -o physaddr=0x20000000,init=0x2F000,bs=1024 none /mnt/pram
```

This command creates a filesystem of 0x2F000 bytes, with a block size of 1024 bytes, and locates it at the physical address 0x20000000. 

To retrieve an existing filesystem, mount the PRAMFS with the `physaddr` parameter that was used in the previous mount. The details of the filesystem such as blocksize and filesystem size are read from the superblock: 

```
# mount -t pramfs -o physaddr=0x20000000 none /mnt/pram
```

Other filesystem parameters are: 

  * `bpi`: specifies the bytes-per-inode ratio. For every `bpi` bytes in the filesystem, an inode is created. 

  * `N`: specifies the number of inodes to allocate in the inode table. If the option is not specified, the bytes-per-inode ratio is used to calculate the number of inodes. 

If the `init` option is not specified, the `bs`, `bpi`, or `N` options are ignored by the mount, since this information is picked up from the existing filesystem. When creating the filesystem, if no option for the inode reservation is specified, by default 5% of the filesystem space is used for the inode table. 

To test the memory protection of PRAMFS, the developers have written a kernel module that attempts to write within the PRAMFS memory with the intention of corrupting the memory space. This causes a kernel protection fault, and, after a reboot, you may re-mount the filesystem to find that the test module was not capable of corrupting the filesystem. 

#### Filesystem Layout

PRAMFS has a simple layout, with the super-block in the first 128 bytes of the RAM block, followed by the inode table, the block usage map, and finally the data blocks. The superblock is 128 bytes long and contains all of the important information, such as filesystem size, block size, etc., needed to remount the filesystem. 

![\[PRAMFS layout\]](https://static.lwn.net/images/pramfs_layout.png)
