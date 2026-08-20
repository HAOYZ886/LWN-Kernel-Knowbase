---
title: "The multi-order radix tree"
url: https://lwn.net/Articles/684864/
date: "April 27, 2016"
category: Radix tree
author: "By Jonathan Corbet April 27, 2016 LSFMM 2016"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
April 27, 2016

* * *

[LSFMM 2016](<https://lwn.net/Articles/lsfmm2016/>)

[Radix trees](<https://lwn.net/Articles/175432/>) have a number of uses in the kernel, the most prominent of which is storing the association between pages in memory and the file blocks that hold their backing store. That particular tree was designed under the assumption that all pages are the same size. When huge pages are used, each single page must be represented by many (typically 512) entries in the radix tree, which is less than fully efficient. Matthew Wilcox has been working on adding multi-order (multiple sizes) support to the radix tree; he described this work in a plenary session at the 2016 Linux Storage, Filesystem, and Memory-Management Summit. 

The core idea behind this work is to allow the radix tree to hold a single entry to represent an entire huge page. This is useful for persistent memory, which is most efficiently managed as huge pages, but it is also desirable for the transparent huge pages mechanism. There have been [![\[Matthew Wilcox\]](https://static.lwn.net/images/conf/2016/lsfmm/MatthewWilcox-sm.jpg)](<https://lwn.net/Articles/684872/>) suggestions that it might assist filesystems in the management of large file blocks, but Matthew is not sure whether that is true or not. Regardless, he wanted developers to know that this functionality is now available. 

The radix tree API has changed little as a result of this work. The low-level functions `__radix_tree_create()` and `__radix_tree_insert()` now have `order` arguments indicating the size of the inserted entry. Code making use of multi-order radix trees may require significant changes, though, so that the code is prepared for the multi-order entries that can be returned by lookup operations. 

Internally, the feature is implemented by tagging pointers with an "indirect" bit to indicate nodes in the tree; if that bit is clear, the pointer refers to user data instead. At the bottom level of the tree, a number of "sibling" entries contain pointers to the canonical entry for the page. Notably, the tree doesn't store the order of the page; users have to get that information via some other means. 

James Bottomley asked whether the tree could be used to detect opportunities for the use of larger pages; Matthew answered that it was probably not the right place for that. Chris Mason said that he once tried to get Btrfs to use stubs in the radix tree as a sort of lock when direct I/O is being performed; that technique might be more easily implemented using the multi-order feature. It could, perhaps, make direct I/O "a bit less of a nightmare." Jan Kara noted that similar things are done to support the DAX direct-access mechanism. He also said that the page cache uses radix-tree entries to mark evicted pages, which could perhaps interfere with other uses. The right solution, perhaps, would be the long-discussed [range-locking mechanism](<https://lwn.net/Articles/548939/>); at this point, though, the session wound down and the idea was not discussed further.
