---
title: Taking control of SSDs with LightNVM
url: https://lwn.net/Articles/641247/
date: "April 22, 2015"
category: "Block layer-Solid-state storage devices; Solid-state storage devices"
author: "By Jonathan Corbet April 22, 2015"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
April 22, 2015

A great deal of work has gone into improving the Linux kernel's block layer so that it can keep up with solid-state storage devices (SSDs). Dealing with SSDs has often been an exercise in frustration, though, for one simple reason: the kernel is not able to manage the storage device directly, but, instead, must talk to a computer embedded in the device that is running some sort of flash translation layer (FTL) software. Developers have often felt that a better job could be done without the FTL getting in the way. Now, it seems, the hardware manufacturers are starting to make direct control easier; a patch set has been posted that aims to enable the kernel to take advantage of this opening. 

The problems with flash translation layers are numerous and well known. Often they are designed to optimize access for one specific filesystem (FAT, for example), a feature that often makes performance worse for other filesystems. Attempts to allow operating systems to communicate usage information to the drives (the discard/TRIM command, which indicates a range of blocks that is not in use, is an example) have led to performance problems and bugs of their own. And an FTL baked into a drive cannot normally be upgraded or fixed when bugs appear. All of these problems would go away if the kernel could just access the low-level storage media directly. 

There are a number of high-end nonvolatile memory (NVM) devices that provide this access now, but there's one catch: each model has its own interface, and sometimes the nature of those interfaces varies wildly. The rest of the kernel, though, cares little about those details; it needs to know a relatively small number of parameters to be able to manage such a device. The required information includes the layout of blocks on the device, some timing details, and not a whole lot more. If there were an abstraction layer in the kernel that provided just the required interface, the task of managing these devices would get easier. 

A candidate for this abstraction layer is [LightNVM](<https://lwn.net/Articles/641358/>), posted by Matias Bjørling. LightNVM is, at its base, a [specification of an interface](<https://docs.google.com/document/d/1kedBY_1-hfkAlqT4EdwY6gz-6UOZbn7kIjWpmBLPNj0/edit>) by which the kernel can access what Matias calls "open-channel SSDs." His implementation adapts the kernel's [NVM Express](<http://www.nvmexpress.org/>) driver to provide the LightNVM interface; the generic block layer code is then adjusted to take advantage of the new capabilities that are provided. 

A LightNVM driver is, to begin with, an ordinary block driver. To get the full performance advantage, it should implement the [multiqueue block interface](<https://lwn.net/Articles/552904/>), though that does not appear to be strictly necessary. On top of the block interface, though, the driver must implement a set of LightNVM-specific APIs, most of which are defined by this structure of function pointers: 

```
struct nvm_dev_ops {
    	nvm_id_fn		*identify;
    	nvm_get_features_fn	*get_features;
    	nvm_set_rsp_fn		*set_responsibility;
    	nvm_get_l2p_tbl_fn	*get_l2p_tbl;
    	nvm_erase_blk_fn	*erase_block;
        };
```

The `identify()` operation identifies the type of the device and, importantly, the number of independent I/O channels it supports; that number affects how many operations can execute in parallel. A call to `get_features()` obtains information about the capabilities of the drive, including whether it can do its own logical-to-physical address mapping, whether the drive performs garbage collection, whether it can perform ECC error correction, and so on. The `set_responsibility()` function tells the drive which features should actually be enabled. The current mapping between logical and physical blocks can be read from the device with `get_l2p_tbl()`. Finally, a call to `erase_block()` will cause a specific erase block to be wiped. 

The code as posted appears to expect that on-drive logical-to-physical mapping will be supported, but that no other features will be present. Adding support for the other features should be an optimization opportunity in the future, especially as drives supporting options like "block move" (which relocates a block on the drive without requiring the host to read and rewrite it) become available. 

At the block-layer level, the patch set provides a mechanism by which the LightNVM code can intercept I/O requests, remap them, and pass the modified request directly to the hardware. For a read request, this task is relatively simple: the logical-to-physical table is consulted to locate the block's address on the drive, then a read is performed from that address. Writes are more complicated, since data in flash cannot be rewritten in place. Instead, the code must find a new location for the block, cause it to be written there, invalidate the copy of the block at the old location, and update the translation maps accordingly. 

In the posted patch set, this work is done by the "RRPC target," a round-robin FTL built into the kernel itself. The wear-leveling algorithm used is fairly simple; the code sequences through erase blocks one after another, relocates any valid sectors found within each erase block, then wipes the block for reuse. It does not even support discard requests at this point. The point is clearly to demonstrate a functioning in-kernel FTL while leaving the optimization opportunities for later. 

There will be a number of such opportunities, but it could take a while to realize many of them. For example, getting the best performance out of such a device requires spreading data across each of the available channel controllers in such a way as to keep them all busy. To an extent, that could be done purely in the FTL, but chances are good that higher performance will result if the filesystem is aware of the device's geometry. Currently there is no API to pass that information up, so, needless to say, no filesystems have that support. 

So LightNVM in its current form is just a start. But it should be enough to test the idea that kernel developers can, in the long run, do a better job of managing flash arrays than the firmware developers who write FTLs have traditionally been able to achieve. 

There is one last question that has not really even been asked yet with regard to this patch set, though. LightNVM is intended to manage nonvolatile memory as if it were a block storage device. But there is [a lot of work](<https://lwn.net/Articles/640113/>) going into the creation of large, nonvolatile-memory devices that are mapped directly into the system's physical address space; from there, that memory can be mapped into a process's virtual address space. While it would be possible to use a kernel layer like LightNVM to do the low-level management for directly mapped devices, that does not appear to be the approach that most manufacturers (and developers) have in mind. So it seems likely that the FTL will remain deeply buried within the hardware for those devices. That could, in the long term, restrict the applicability of block-oriented subsystems like LightNVM.
