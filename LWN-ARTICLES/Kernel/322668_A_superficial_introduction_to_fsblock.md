---
title: A superficial introduction to fsblock
url: https://lwn.net/Articles/322668/
date: "March 11, 2009"
category: Block layer; Filesystems; fsblock
author: "By Jonathan Corbet March 11, 2009"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
March 11, 2009

Many kernel developers may work through their entire career without encountering a buffer_head structure. But the buffer head (often called "bh") sits at the core of the kernel's memory management and filesystem layers. Simply put, a bh maintains a mapping between a specific page (or portion thereof) in RAM and its corresponding block on disk. In the 2.4 days, the bh structure was also a key part of the block I/O layer, but 2.6 broke that particular association. That notwithstanding, the lowly, much-maligned bh still plays a crucial role in contemporary kernels. 

Why "much-maligned"? Buffer heads are difficult to manage, to the point that they can create significant memory pressure on some systems. They deal in very small units of I/O (512 bytes), so you need a pile of them to represent even a single page. And there is a certain sense of antiquity that one encounters when dealing with them; the buffer head code is some of the oldest code in the core kernel. But it is important and tricky code, so few developers dare to try to improve it. 

Nick Piggin is the daring type. But Nick, too, is not trying to improve the bh layer; instead, he would like to replace it outright. The result is an intimidating set of large patches known as "fsblock." This code was first [posted](<http://lwn.net/Articles/239621/>) in 2007, making it fairly young by the standards of memory-management patches. This patch set was [reposted](<http://lwn.net/Articles/321390/>) in early March; it has shown a number of improvements on the way. Nick says ""I'm pretty intent on getting it merged sooner or later"", so we'll likely be seeing more of this code in the future. 

The core data structure is `struct fsblock`, which represents one block: 

```
struct fsblock {
    	unsigned int	flags;
    	unsigned int	count;
    
        #ifdef BDFLUSH_FLUSHING
    	struct rb_node	block_node;
        #endif
    	sector_t	block_nr;
    	void		*private;
    	struct page	*page;
        };
```

This structure, notes Nick, is about 1/3 the size of [`struct buffer_head`](<https://lwn.net/Articles/322674/>), but it serves roughly the same purpose: tracking the association between an in-memory block (found in `page`) and its on-disk version, indexed by `block_nr`. The `flags` field describes the state of this block: whether it's up-to-date (memory and disk versions match), locked, dirty, in writeback, etc. Some of these flags (the dirty state, for example) match the state stored with the in-memory page; the fsblock layer (unlike the buffer_head code) takes great care to keep those flags in sync. 

There are a couple of interesting flags in the `fsblock` structure which one does not find associated with buffer heads. One of them is not a flag at all: `BL_bits_mask` describes a subfield giving the size of the block. In fsblock, "blocks" are not limited to the standard 512-byte sector size; they can, in fact, even be larger than a page. These "superpage" blocks have been on some filesystem developers' wish lists for some time; they would make it easy to create filesystems with large blocks which, in turn, would perform better in a number of situations. But the superpage feature may be removed for any initial merge of fsblock in an attempt to make the code easier to understand and review. Besides, large blocks are a bit of a controversial topic, so it makes sense to address that issue separately. 

The `flags` field also holds a flag called `BL_metadata`; this flag indicates a block which holds filesystem metadata instead of file data. In this case, the block is actually part of a larger structure which (edited slightly) looks like this: 

```
struct fsblock_meta {
    	struct fsblock block;
    	union {
        #ifdef VMAP_CACHE
    	    /* filesystems using vmap APIs should not use ->data */
    	    struct vmap_cache_entry *vce;
        #endif
    	    /*
    	     * data is a direct mapping to the block device data, used by
    	     * "intermediate" mode filesystems.
    	     */
    	    char *data;
    	};
        };
```

In short, this structure makes it easy for filesystem code to deal directly with metadata blocks. Finally, the `fsblock_sb` structure ties a filesystem superblock into the fsblock subsystem. 

A filesystem can, at mount time, set things up with a call to: 

```
int fsblock_register_super(struct super_block *sb, 
                                   struct fsblock_sb *fsb_sb);
```

The superblock can then be read in with a call to `sb_mbread()`: 

```
struct fsblock_meta *sb_mbread(struct fsblock_sb *fsb_sb, 
                                       sector_t blocknr);
```

There's only one little problem: before fsblock can perform block I/O operations, it must have access to the superblock. So, thus far, filesystems which have been converted to fsblock must still use the buffer head API to read the superblock. One assumes that this little glitch will be taken care of at some point. 

A tour of the full fsblock API would require a few articles - it is a lot of code. Hopefully a quick overview will provide a sense for how it all works. To start with, blocks are reference-counted objects in fsblock, so there is the usual set of functions for incrementing and decrementing the counts: 

```
void block_get(struct fsblock *block);
        void block_put(struct fsblock *block);
        void mblock_get(struct fsblock_meta *block);
        void mblock_put(struct fsblock_meta *block);
```

There's a whole set of functions for performing I/O on blocks and metadata blocks; some of these are: 

```
struct fsblock_meta *mbread(struct fsblock_sb *fsb_sb, sector_t blocknr, 
        	   		        unsigned int size);
        int mblock_read_sync(struct fsblock_meta *mb);
        int sync_block(struct fsblock *block);
```

Note that, while there are a number of functions for reading blocks, there are fewer write functions. Instead, code will use a function like `set_block_dirty()` or `mark_mblock_dirty()`, then leave it up to the memory management code to decide when the actual I/O should take place. 

There is a lot more than this to fsblock, including functions to lock blocks, look up in-memory blocks, perform page I/O, truncate pages, implement `mmap()`, and more. One assumes that Nick will certainly write exhaustive documentation for this API sometime soon. 

Beyond that little documentation task, there are a few other things to do, including supporting direct I/O and fixing a number of known bugs. But, even now, fsblock seems to have a lot of potential; it updates the old buffer head API in a way which is more efficient and more robust. It also appears to perform better with the ext2 filesystem - a fact which appears to be surprising to Nick. So something like fsblock will almost certainly be merged sooner or later. A lot could happen in the mean time, though. Core memory-management-related patches like this are notoriously slow to get through the merging process, and, despite its age, fsblock has not seen a great deal of review to date. So there's likely to be plenty of time and opportunity for other developers to find things to disagree with before fsblock hits the mainline.
